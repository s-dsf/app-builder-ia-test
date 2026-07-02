# backend_engineer

**Projet :** app voyage 3
**Mis à jour :** 01/07/2026

---

— BACKEND ENGINEER

# Livrable Backend Engineer — Travel Companion App
## MVP B2C · Next.js API Routes · Supabase · Anthropic API

---

## 0. NOTE DE CADRAGE TECHNIQUE

**Adaptation stack** : Les livrables précédents prévoient Node.js/Express + PostgreSQL + Redis. En V1 avec Next.js API Routes + Supabase, les équivalences sont :

| Architecture prévue | Implémentation V1 |
|---|---|
| Node.js Express | Next.js API Routes (`/app/api/`) |
| PostgreSQL | Supabase PostgreSQL |
| Redis (cache) | Supabase Edge Cache + cache in-memory sur routes |
| S3-compatible | Supabase Storage |
| JWT custom | Supabase Auth (JWT inclus) |
| OAuth Google/Apple | Supabase Auth providers |

**Ce que Supabase nous donne gratuitement** : Auth complète, RLS natif PostgreSQL, Storage, Realtime, Edge Functions si besoin. Le backend se concentre sur la logique métier complexe que Supabase ne couvre pas nativement.

**APIs tierces encapsulées côté serveur uniquement** :
- DeepL API → traduction
- Fixer.io / Open Exchange Rates → devises
- Timezone API → fuseaux horaires

> ⚠️ Aucune clé API tierce n'est jamais exposée côté client. Tous les appels tierces transitent par les API Routes Next.js.

---

## 1. SCHÉMA DE BASE DE DONNÉES

### 1.1 Diagramme des relations

```
users (géré par Supabase Auth)
  │
  ├──► profiles (1:1)
  │
  ├──► trips (1:N) [créateur]
  │      │
  │      ├──► trip_participants (N:M) ◄── users
  │      │
  │      ├──► trip_steps (1:N)
  │      │      └──► step_documents (N:M) ◄── documents
  │      │
  │      ├──► documents (1:N)
  │      │
  │      ├──► memories (1:N)
  │      │      └──► memory_media (1:N)
  │      │
  │      └──► shared_albums (1:N)
  │             └──► shared_album_memories (N:M) ◄── memories
  │
  └──► notifications (1:N)
```

### 1.2 Schéma SQL complet

```sql
-- ============================================
-- EXTENSION & CONFIGURATION
-- ============================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm"; -- recherche texte

-- ============================================
-- TABLE: profiles
-- Extension de auth.users (Supabase)
-- ============================================
CREATE TABLE public.profiles (
  id              UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name    TEXT,
  avatar_url      TEXT,
  preferred_lang  TEXT NOT NULL DEFAULT 'fr',       -- langue UI
  preferred_currency TEXT NOT NULL DEFAULT 'EUR',   -- devise par défaut
  timezone        TEXT NOT NULL DEFAULT 'Europe/Paris',
  push_token      TEXT,                              -- FCM/APNs token device
  push_enabled    BOOLEAN NOT NULL DEFAULT true,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: trips
-- Un voyage = espace central de toutes les données
-- ============================================
CREATE TABLE public.trips (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  owner_id        UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  name            TEXT NOT NULL,
  destination     TEXT NOT NULL,                     -- nom lisible "Bali, Indonésie"
  destination_country_code CHAR(2),                  -- ISO 3166-1 alpha-2 "ID"
  destination_currency TEXT,                         -- devise locale "IDR"
  destination_timezone TEXT,                         -- "Asia/Makassar"
  destination_language TEXT,                         -- langue principale "id" (ISO 639-1)
  cover_image_url TEXT,
  start_date      DATE NOT NULL,
  end_date        DATE NOT NULL,
  status          TEXT NOT NULL DEFAULT 'upcoming'   -- upcoming | active | past
                  CHECK (status IN ('upcoming', 'active', 'past')),
  is_archived     BOOLEAN NOT NULL DEFAULT false,
  notes           TEXT,                              -- notes générales du voyage
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT trips_dates_check CHECK (end_date >= start_date)
);

-- ============================================
-- TABLE: trip_participants
-- Participants invités à un voyage
-- ============================================
CREATE TABLE public.trip_participants (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id         UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  user_id         UUID REFERENCES auth.users(id) ON DELETE SET NULL,
  email           TEXT NOT NULL,                     -- invitation par email
  display_name    TEXT,                              -- nom si user non inscrit
  role            TEXT NOT NULL DEFAULT 'viewer'
                  CHECK (role IN ('admin', 'editor', 'viewer')),
  status          TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending', 'accepted', 'declined')),
  invited_by      UUID NOT NULL REFERENCES auth.users(id),
  invited_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  responded_at    TIMESTAMPTZ,

  UNIQUE(trip_id, email)
);

-- ============================================
-- TABLE: trip_steps
-- Étapes de l'itinéraire (chronologiques)
-- ============================================
CREATE TABLE public.trip_steps (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id         UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  title           TEXT NOT NULL,
  step_type       TEXT NOT NULL DEFAULT 'activity'
                  CHECK (step_type IN (
                    'flight', 'accommodation', 'activity',
                    'transport', 'restaurant', 'other'
                  )),
  step_date       DATE NOT NULL,
  start_time      TIME,                              -- heure locale destination
  end_time        TIME,
  location_name   TEXT,
  location_lat    NUMERIC(10, 7),
  location_lng    NUMERIC(10, 7),
  location_address TEXT,
  notes           TEXT,
  reminder_enabled BOOLEAN NOT NULL DEFAULT false,
  reminder_offset_minutes INT DEFAULT 120,           -- délai avant étape (120 = 2h)
  reminder_sent   BOOLEAN NOT NULL DEFAULT false,
  sort_order      INT NOT NULL DEFAULT 0,
  created_by      UUID NOT NULL REFERENCES auth.users(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: documents
-- Documents associés à un voyage
-- ============================================
CREATE TABLE public.documents (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id         UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  step_id         UUID REFERENCES public.trip_steps(id) ON DELETE SET NULL,
  uploaded_by     UUID NOT NULL REFERENCES auth.users(id),
  file_name       TEXT NOT NULL,
  file_type       TEXT NOT NULL,                     -- MIME type
  file_size_bytes BIGINT NOT NULL,
  storage_path    TEXT NOT NULL,                     -- chemin Supabase Storage
  category        TEXT NOT NULL DEFAULT 'other'
                  CHECK (category IN (
                    'flight', 'accommodation', 'activity',
                    'transport', 'identity', 'insurance', 'other'
                  )),
  is_cached_offline BOOLEAN NOT NULL DEFAULT false,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT documents_size_check CHECK (file_size_bytes <= 20971520) -- 20 Mo max
);

-- ============================================
-- TABLE: memories
-- Souvenirs (photo, note, lieu)
-- ============================================
CREATE TABLE public.memories (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id         UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  created_by      UUID NOT NULL REFERENCES auth.users(id),
  caption         TEXT,
  memory_date     DATE,
  location_name   TEXT,
  location_lat    NUMERIC(10, 7),
  location_lng    NUMERIC(10, 7),
  is_favorite     BOOLEAN NOT NULL DEFAULT false,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: memory_media
-- Médias attachés à un souvenir
-- ============================================
CREATE TABLE public.memory_media (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  memory_id       UUID NOT NULL REFERENCES public.memories(id) ON DELETE CASCADE,
  storage_path    TEXT NOT NULL,
  media_type      TEXT NOT NULL DEFAULT 'image'
                  CHECK (media_type IN ('image', 'video')),
  file_size_bytes BIGINT NOT NULL,
  width           INT,
  height          INT,
  sort_order      INT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: shared_albums
-- Albums partagés publiquement (lien unique)
-- ============================================
CREATE TABLE public.shared_albums (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id         UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  created_by      UUID NOT NULL REFERENCES auth.users(id),
  title           TEXT,
  slug            TEXT UNIQUE NOT NULL,              -- identifiant URL unique
  is_active       BOOLEAN NOT NULL DEFAULT true,
  password_hash   TEXT,                              -- protection optionnelle
  expires_at      TIMESTAMPTZ,                       -- expiration optionnelle
  view_count      INT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: shared_album_memories
-- Association album ↔ souvenirs sélectionnés
-- ============================================
CREATE TABLE public.shared_album_memories (
  album_id        UUID NOT NULL REFERENCES public.shared_albums(id) ON DELETE CASCADE,
  memory_id       UUID NOT NULL REFERENCES public.memories(id) ON DELETE CASCADE,
  sort_order      INT NOT NULL DEFAULT 0,
  PRIMARY KEY (album_id, memory_id)
);

-- ============================================
-- TABLE: currency_cache
-- Cache des taux de change (évite appels API répétés)
-- ============================================
CREATE TABLE public.currency_cache (
  base_currency   CHAR(3) PRIMARY KEY,
  rates           JSONB NOT NULL,                    -- { "EUR": 1.0, "USD": 1.08, ... }
  fetched_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: translation_cache
-- Cache optionnel traductions fréquentes
-- ============================================
CREATE TABLE public.translation_cache (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  source_text     TEXT NOT NULL,
  source_lang     CHAR(5) NOT NULL,
  target_lang     CHAR(5) NOT NULL,
  translated_text TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  UNIQUE(source_text, source_lang, target_lang)
);

-- ============================================
-- INDEX
-- ============================================
CREATE INDEX idx_trips_owner_id ON public.trips(owner_id);
CREATE INDEX idx_trips_status ON public.trips(status) WHERE is_archived = false;
CREATE INDEX idx_trip_steps_trip_date ON public.trip_steps(trip_id, step_date);
CREATE INDEX idx_trip_steps_reminder ON public.trip_steps(step_date, reminder_enabled)
  WHERE reminder_sent = false AND reminder_enabled = true;
CREATE INDEX idx_documents_trip_id ON public.documents(trip_id);
CREATE INDEX idx_documents_step_id ON public.documents(step_id);
CREATE INDEX idx_memories_trip_id ON public.memories(trip_id);
CREATE INDEX idx_trip_participants_email ON public.trip_participants(email);
CREATE INDEX idx_trip_participants_user_id ON public.trip_participants(user_id);
CREATE INDEX idx_shared_albums_slug ON public.shared_albums(slug) WHERE is_active = true;
```

---

### 1.3 Politique RLS (Row Level Security)

```sql
-- ============================================
-- ACTIVATION RLS SUR TOUTES LES TABLES
-- ============================================
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.trips ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.trip_participants ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.trip_steps ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.memories ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.memory_media ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.shared_albums ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.shared_album_memories ENABLE ROW LEVEL SECURITY;

-- ============================================
-- HELPER FUNCTION : vérifie l'accès à un voyage
-- ============================================
CREATE OR REPLACE FUNCTION public.has_trip_access(trip_uuid UUID)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1 FROM public.trips t
    WHERE t.id = trip_uuid AND t.owner_id = auth.uid()
    UNION
    SELECT 1 FROM public.trip_participants tp
    WHERE tp.trip_id = trip_uuid
      AND tp.user_id = auth.uid()
      AND tp.status = 'accepted'
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- ============================================
-- POLICIES : profiles
-- ============================================
CREATE POLICY "Profil visible par son propriétaire"
  ON public.profiles FOR SELECT
  USING (id = auth.uid());

CREATE POLICY "Profil modifiable par son propriétaire"
  ON public.profiles FOR UPDATE
  USING (id = auth.uid());

CREATE POLICY "Profil créé à l'inscription"
  ON public.profiles FOR INSERT
  WITH CHECK (id = auth.uid());

-- ============================================
-- POLICIES : trips
-- ============================================
CREATE POLICY "Voyages accessibles aux membres"
  ON public.trips FOR SELECT
  USING (public.has_trip_access(id));

CREATE POLICY "Voyage créé par l'utilisateur connecté"
  ON public.trips FOR INSERT
  WITH CHECK (owner_id = auth.uid());

CREATE POLICY "Voyage modifiable par owner ou admin"
  ON public.trips FOR UPDATE
  USING (
    owner_id = auth.uid()
    OR EXISTS (
      SELECT 1 FROM public.trip_participants
      WHERE trip_id = id
        AND user_id = auth.uid()
        AND role = 'admin'
        AND status = 'accepted'
    )
  );

CREATE POLICY "Voyage supprimable par owner uniquement"
  ON public.trips FOR DELETE
  USING (owner_id = auth.uid());

-- ============================================
-- POLICIES : trip_steps
-- ============================================
CREATE POLICY "Étapes visibles par membres du voyage"
  ON public.trip_steps FOR SELECT
  USING (public.has_trip_access(trip_id));

CREATE POLICY "Étapes créées par editors ou admins"
  ON public.trip_steps FOR INSERT
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM public.trips WHERE id = trip_id AND owner_id = auth.uid()
    )
    OR EXISTS (
      SELECT 1 FROM public.trip_participants

— BACKEND ENGINEER

# Livrable Backend Engineer — Travel Companion App
## MVP B2C · Next.js API Routes · Supabase · Anthropic Claude

---

> **Note de cadrage stack** : Conformément aux livrables disponibles et à la décision d'utiliser Supabase, ce livrable cible **Next.js API Routes** comme couche API (compatible avec le frontend Next.js 14), **Supabase** pour la base de données PostgreSQL + Auth + Storage, et l'**API Anthropic Claude** pour les fonctionnalités d'IA éventuelles (assistance à la création d'itinéraire, analyse de documents). Le schéma de données s'appuie sur la base posée dans les livrables précédents et est ici finalisé et complété.

---

## 1. SCHÉMA DE BASE DE DONNÉES COMPLET

### 1.1 Tables manquantes / à compléter

Les tables suivantes complètent ce qui a été initié dans les livrables précédents.

```sql
-- ============================================
-- TABLE: profiles
-- Extension du user Supabase Auth
-- ============================================
CREATE TABLE public.profiles (
  id                UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name      TEXT,
  avatar_url        TEXT,
  default_currency  CHAR(3) NOT NULL DEFAULT 'EUR',
  default_language  CHAR(5) NOT NULL DEFAULT 'fr',
  timezone          TEXT NOT NULL DEFAULT 'Europe/Paris',
  push_token        TEXT,                          -- FCM / APNs token
  push_token_updated_at TIMESTAMPTZ,
  onboarding_done   BOOLEAN NOT NULL DEFAULT false,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: trips
-- Voyage principal
-- ============================================
CREATE TABLE public.trips (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  owner_id          UUID NOT NULL REFERENCES auth.users(id),
  name              TEXT NOT NULL,
  destination       TEXT NOT NULL,
  destination_lat   NUMERIC(10, 7),
  destination_lng   NUMERIC(10, 7),
  country_code      CHAR(2),                       -- ISO 3166-1 alpha-2
  currency_local    CHAR(3),                       -- devise locale destination
  language_local    CHAR(5),                       -- langue locale destination
  timezone_local    TEXT,                          -- fuseau horaire destination
  start_date        DATE NOT NULL,
  end_date          DATE NOT NULL,
  cover_image_path  TEXT,
  status            TEXT NOT NULL DEFAULT 'upcoming'
                    CHECK (status IN ('upcoming', 'ongoing', 'completed')),
  is_archived       BOOLEAN NOT NULL DEFAULT false,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT trips_dates_check CHECK (end_date >= start_date)
);

-- ============================================
-- TABLE: trip_participants
-- Participants invités à un voyage
-- ============================================
CREATE TABLE public.trip_participants (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id           UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  user_id           UUID REFERENCES auth.users(id),   -- NULL si invitation en attente
  email             TEXT NOT NULL,
  role              TEXT NOT NULL DEFAULT 'viewer'
                    CHECK (role IN ('admin', 'editor', 'viewer')),
  status            TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'accepted', 'declined')),
  invitation_token  TEXT UNIQUE,                   -- token d'invitation par email
  invited_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  accepted_at       TIMESTAMPTZ,

  UNIQUE(trip_id, email)
);

-- ============================================
-- TABLE: trip_steps
-- Étapes de l'itinéraire
-- ============================================
CREATE TABLE public.trip_steps (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id           UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  title             TEXT NOT NULL,
  step_type         TEXT NOT NULL DEFAULT 'activity'
                    CHECK (step_type IN (
                      'flight', 'accommodation', 'activity',
                      'transport', 'restaurant', 'other'
                    )),
  step_date         DATE NOT NULL,
  start_time        TIME,
  end_time          TIME,
  location_name     TEXT,
  location_lat      NUMERIC(10, 7),
  location_lng      NUMERIC(10, 7),
  location_address  TEXT,
  notes             TEXT,
  booking_ref       TEXT,
  reminder_enabled  BOOLEAN NOT NULL DEFAULT false,
  reminder_offset   INT NOT NULL DEFAULT 120,      -- minutes avant l'étape
  reminder_sent     BOOLEAN NOT NULL DEFAULT false,
  sort_order        INT NOT NULL DEFAULT 0,
  created_by        UUID NOT NULL REFERENCES auth.users(id),
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: documents
-- Fichiers importés (PDF, images)
-- ============================================
CREATE TABLE public.documents (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id           UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  step_id           UUID REFERENCES public.trip_steps(id) ON DELETE SET NULL,
  uploaded_by       UUID NOT NULL REFERENCES auth.users(id),
  file_name         TEXT NOT NULL,
  file_size_bytes   BIGINT NOT NULL,
  mime_type         TEXT NOT NULL,
  storage_path      TEXT NOT NULL,
  category          TEXT NOT NULL DEFAULT 'other'
                    CHECK (category IN (
                      'flight', 'accommodation', 'activity',
                      'transport', 'identity', 'insurance', 'other'
                    )),
  is_cached_offline BOOLEAN NOT NULL DEFAULT false,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  CONSTRAINT documents_size_check CHECK (file_size_bytes <= 20971520)
);

-- ============================================
-- TABLE: memories
-- ============================================
CREATE TABLE public.memories (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id           UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  created_by        UUID NOT NULL REFERENCES auth.users(id),
  caption           TEXT,
  memory_date       DATE,
  location_name     TEXT,
  location_lat      NUMERIC(10, 7),
  location_lng      NUMERIC(10, 7),
  is_favorite       BOOLEAN NOT NULL DEFAULT false,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: memory_media
-- ============================================
CREATE TABLE public.memory_media (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  memory_id         UUID NOT NULL REFERENCES public.memories(id) ON DELETE CASCADE,
  storage_path      TEXT NOT NULL,
  media_type        TEXT NOT NULL DEFAULT 'image'
                    CHECK (media_type IN ('image', 'video')),
  file_size_bytes   BIGINT NOT NULL,
  width             INT,
  height            INT,
  sort_order        INT NOT NULL DEFAULT 0,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: shared_albums
-- ============================================
CREATE TABLE public.shared_albums (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id           UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  created_by        UUID NOT NULL REFERENCES auth.users(id),
  title             TEXT,
  slug              TEXT UNIQUE NOT NULL,
  is_active         BOOLEAN NOT NULL DEFAULT true,
  password_hash     TEXT,
  expires_at        TIMESTAMPTZ,
  view_count        INT NOT NULL DEFAULT 0,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: shared_album_memories
-- ============================================
CREATE TABLE public.shared_album_memories (
  album_id          UUID NOT NULL REFERENCES public.shared_albums(id) ON DELETE CASCADE,
  memory_id         UUID NOT NULL REFERENCES public.memories(id) ON DELETE CASCADE,
  sort_order        INT NOT NULL DEFAULT 0,
  PRIMARY KEY (album_id, memory_id)
);

-- ============================================
-- TABLE: currency_cache
-- ============================================
CREATE TABLE public.currency_cache (
  base_currency     CHAR(3) PRIMARY KEY,
  rates             JSONB NOT NULL,
  fetched_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================
-- TABLE: translation_cache
-- ============================================
CREATE TABLE public.translation_cache (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  source_text_hash  TEXT NOT NULL,               -- SHA-256 du texte source (perf)
  source_lang       CHAR(5) NOT NULL,
  target_lang       CHAR(5) NOT NULL,
  translated_text   TEXT NOT NULL,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  UNIQUE(source_text_hash, source_lang, target_lang)
);

-- ============================================
-- TABLE: notification_logs
-- Historique des notifications envoyées
-- ============================================
CREATE TABLE public.notification_logs (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id           UUID NOT NULL REFERENCES auth.users(id),
  step_id           UUID REFERENCES public.trip_steps(id) ON DELETE SET NULL,
  notification_type TEXT NOT NULL DEFAULT 'reminder'
                    CHECK (notification_type IN ('reminder', 'invitation', 'system')),
  title             TEXT NOT NULL,
  body              TEXT NOT NULL,
  sent_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  delivery_status   TEXT NOT NULL DEFAULT 'sent'
                    CHECK (delivery_status IN ('sent', 'delivered', 'failed'))
);

-- ============================================
-- INDEX COMPLETS
-- ============================================
CREATE INDEX idx_trips_owner_id        ON public.trips(owner_id);
CREATE INDEX idx_trips_status          ON public.trips(status) WHERE is_archived = false;
CREATE INDEX idx_trips_dates           ON public.trips(start_date, end_date);
CREATE INDEX idx_trip_steps_trip_date  ON public.trip_steps(trip_id, step_date);
CREATE INDEX idx_trip_steps_reminder   ON public.trip_steps(step_date, reminder_enabled)
  WHERE reminder_sent = false AND reminder_enabled = true;
CREATE INDEX idx_documents_trip_id     ON public.documents(trip_id);
CREATE INDEX idx_documents_step_id     ON public.documents(step_id);
CREATE INDEX idx_memories_trip_id      ON public.memories(trip_id);
CREATE INDEX idx_memory_media_memory   ON public.memory_media(memory_id, sort_order);
CREATE INDEX idx_participants_email    ON public.trip_participants(email);
CREATE INDEX idx_participants_user_id  ON public.trip_participants(user_id);
CREATE INDEX idx_participants_token    ON public.trip_participants(invitation_token)
  WHERE invitation_token IS NOT NULL;
CREATE INDEX idx_shared_albums_slug    ON public.shared_albums(slug) WHERE is_active = true;
CREATE INDEX idx_notif_logs_user       ON public.notification_logs(user_id, sent_at DESC);
CREATE INDEX idx_translation_hash      ON public.translation_cache(source_text_hash, source_lang, target_lang);
```

---

### 1.2 Politique RLS — Suite et complément

```sql
-- ============================================
-- HELPER FUNCTIONS
-- ============================================

-- Vérifie l'accès membre à un voyage (owner OU participant accepté)
CREATE OR REPLACE FUNCTION public.has_trip_access(trip_uuid UUID)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1 FROM public.trips
    WHERE id = trip_uuid AND owner_id = auth.uid()
  ) OR EXISTS (
    SELECT 1 FROM public.trip_participants
    WHERE trip_id = trip_uuid
      AND user_id = auth.uid()
      AND status = 'accepted'
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

-- Vérifie le droit d'écriture (owner, admin ou editor)
CREATE OR REPLACE FUNCTION public.can_edit_trip(trip_uuid UUID)
RETURNS BOOLEAN AS $$
BEGIN
  RETURN EXISTS (
    SELECT 1 FROM public.trips
    WHERE id = trip_uuid AND owner_id = auth.uid()
  ) OR EXISTS (
    SELECT 1 FROM public.trip_participants
    WHERE trip_id = trip_uuid
      AND user_id = auth.uid()
      AND role IN ('admin', 'editor')
      AND status = 'accepted'
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER STABLE;

-- ============================================
-- POLICIES : trip_steps (suite des policies précédentes)
-- ============================================
CREATE POLICY "Étapes créées par editors ou admins"
  ON public.trip_steps FOR INSERT
  WITH CHECK (public.can_edit_trip(trip_id));

CREATE POLICY "Étapes modifiables par editors ou admins"
  ON public.trip_steps FOR UPDATE
  USING (public.can_edit_trip(trip_id));

CREATE POLICY "Étapes supprimables par editors ou admins"
  ON public.trip_steps FOR DELETE
  USING (public.can_edit_trip(trip_id));

-- ============================================
-- POLICIES : documents
-- ============================================
CREATE POLICY "Documents visibles par membres"
  ON public.documents FOR SELECT
  USING (public.has_trip_access(trip_id));

CREATE POLICY "Documents uploadés par editors ou admins"
  ON public.documents FOR INSERT
  WITH CHECK (public.can_edit_trip(trip_id) AND uploaded_by = auth.uid());

CREATE POLICY "Documents supprimables par uploadeur ou admin"
  ON public.documents FOR DELETE
  USING (
    uploaded_by = auth.uid()
    OR EXISTS (
      SELECT 1 FROM public.trips WHERE id = trip_id AND owner_id = auth.uid()
    )
    OR EXISTS (
      SELECT 1 FROM public.trip_participants
      WHERE trip_id = documents.trip_id
        AND user_id = auth.uid()
        AND role = 'admin'
        AND status = 'accepted'
    )
  );

-- ============================================
-- POLICIES : memories
-- ============================================
CREATE POLICY "Souvenirs visibles par membres"
  ON public.memories FOR SELECT
  USING (public.has_trip_access(trip_id));

CREATE POLICY "Souvenir créé par n'importe quel membre"
  ON public.memories FOR INSERT
  WITH CHECK (public.has_trip_access(trip_id) AND created_by = auth.uid());

CREATE POLICY "Souvenir modifiable par son créateur"
  ON public.memories FOR UPDATE
  USING (created_by = auth.uid());

CREATE POLICY "Souvenir supprimable par créateur ou owner voyage"
  ON public.memories FOR DELETE
  USING (
    created_by = auth.uid()
    OR EXISTS (
      SELECT 1 FROM public.trips WHERE id = trip_id AND owner_id = auth.uid()
    )
  );

-- ============================================
-- POLICIES : memory_media
-- ============================================
CREATE POLICY "Médias visibles si accès au souvenir"
  ON public.memory_media FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM public.memories m
      WHERE m.id = memory_id AND public.has_trip_access(m.trip_id)
    )
  );

CREATE POLICY "Médias créés par créateur du souvenir"
  ON public.memory_media FOR INSERT
  WITH CHECK (
    EXISTS (
      SELECT 1 FROM public.memories m
      WHERE m.id = memory

— BACKEND ENGINEER

# Livrable Backend Engineer — Travel Companion App
## MVP B2C · Next.js API Routes · Supabase · Anthropic

---

> **Note de cadrage** : Ce livrable s'appuie sur la stack définie par l'Architecte et le Tech Lead, adaptée au contexte d'App Builder IA : **Next.js API Routes** pour le backend, **Supabase** (PostgreSQL + Auth + Storage + RLS) comme infrastructure de données, et **l'API Anthropic** pour toute fonctionnalité d'intelligence (assistance, extraction de données de documents, suggestions d'itinéraire). Le schéma de base de données et les policies RLS ont été définis dans les livrables précédents — ce document les complète et se concentre sur l'architecture des services et les routes critiques.

---

## 1. SCHÉMA DE BASE DE DONNÉES — RÉCAPITULATIF ET COMPLÉMENTS

### 1.1 Tables définies (livrables précédents)

Les tables suivantes ont été définies et ne sont pas répétées :
- `users` (géré par `auth.users` Supabase)
- `trips`
- `trip_steps`
- `trip_participants`
- `documents`
- `memories`
- `memory_media`
- `shared_albums`
- `shared_album_memories`
- `currency_cache`
- `translation_cache`
- `notification_logs`

### 1.2 Tables manquantes à créer

```sql
-- ============================================
-- TABLE: user_profiles
-- Extension de auth.users pour les données applicatives
-- ============================================
CREATE TABLE public.user_profiles (
  id                UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name      TEXT,
  avatar_url        TEXT,
  default_currency  CHAR(3) NOT NULL DEFAULT 'EUR',
  default_language  CHAR(5) NOT NULL DEFAULT 'fr',
  timezone          TEXT NOT NULL DEFAULT 'Europe/Paris',
  push_token        TEXT,                           -- FCM/APNs token
  push_token_updated_at TIMESTAMPTZ,
  onboarding_completed BOOLEAN NOT NULL DEFAULT false,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- RLS user_profiles
ALTER TABLE public.user_profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Profil lisible par son propriétaire"
  ON public.user_profiles FOR SELECT
  USING (id = auth.uid());

CREATE POLICY "Profil modifiable par son propriétaire"
  ON public.user_profiles FOR UPDATE
  USING (id = auth.uid());

CREATE POLICY "Profil créé par son propriétaire"
  ON public.user_profiles FOR INSERT
  WITH CHECK (id = auth.uid());

-- ============================================
-- TABLE: step_reminders
-- Planification des rappels (découplé de trip_steps)
-- ============================================
CREATE TABLE public.step_reminders (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  step_id           UUID NOT NULL REFERENCES public.trip_steps(id) ON DELETE CASCADE,
  user_id           UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  remind_at         TIMESTAMPTZ NOT NULL,           -- timestamp calculé (step_date - offset)
  offset_minutes    INT NOT NULL DEFAULT 120,       -- 120 = 2h avant
  status            TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'sent', 'cancelled', 'failed')),
  sent_at           TIMESTAMPTZ,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

  UNIQUE(step_id, user_id, offset_minutes)
);

ALTER TABLE public.step_reminders ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Rappels visibles par leur propriétaire"
  ON public.step_reminders FOR SELECT
  USING (user_id = auth.uid());

CREATE POLICY "Rappels créés par leur propriétaire"
  ON public.step_reminders FOR INSERT
  WITH CHECK (user_id = auth.uid());

CREATE POLICY "Rappels modifiables par leur propriétaire"
  ON public.step_reminders FOR UPDATE
  USING (user_id = auth.uid());

CREATE POLICY "Rappels supprimables par leur propriétaire"
  ON public.step_reminders FOR DELETE
  USING (user_id = auth.uid());

CREATE INDEX idx_step_reminders_pending
  ON public.step_reminders(remind_at)
  WHERE status = 'pending';

-- ============================================
-- TABLE: ai_suggestions
-- Cache des suggestions générées par Anthropic
-- ============================================
CREATE TABLE public.ai_suggestions (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id           UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  suggestion_type   TEXT NOT NULL
                    CHECK (suggestion_type IN ('itinerary', 'document_parse', 'step_enrich')),
  input_hash        TEXT NOT NULL,                  -- hash SHA-256 du prompt/input
  response_payload  JSONB NOT NULL,
  model_version     TEXT NOT NULL DEFAULT 'claude-3-5-haiku-20241022',
  tokens_used       INT,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at        TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '7 days',

  UNIQUE(trip_id, suggestion_type, input_hash)
);

ALTER TABLE public.ai_suggestions ENABLE ROW LEVEL SECURITY;

-- Seul le backend (service role) écrit dans cette table
-- Les utilisateurs peuvent lire les suggestions de leurs voyages
CREATE POLICY "Suggestions lisibles par membres du voyage"
  ON public.ai_suggestions FOR SELECT
  USING (public.has_trip_access(trip_id));

-- ============================================
-- TABLE: document_extractions
-- Résultats d'extraction IA sur les documents importés
-- ============================================
CREATE TABLE public.document_extractions (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  document_id       UUID NOT NULL REFERENCES public.documents(id) ON DELETE CASCADE,
  extracted_data    JSONB NOT NULL,  -- {type, dates, locations, references, amounts}
  confidence_score  FLOAT,
  model_version     TEXT NOT NULL,
  status            TEXT NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'completed', 'failed')),
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE public.document_extractions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Extractions lisibles par membres du voyage"
  ON public.document_extractions FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM public.documents d
      WHERE d.id = document_id AND public.has_trip_access(d.trip_id)
    )
  );

-- ============================================
-- TRIGGER: auto-création user_profile à l'inscription
-- ============================================
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.user_profiles (id, display_name, avatar_url)
  VALUES (
    NEW.id,
    COALESCE(NEW.raw_user_meta_data->>'full_name', NEW.email),
    NEW.raw_user_meta_data->>'avatar_url'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();

-- ============================================
-- TRIGGER: updated_at automatique
-- ============================================
CREATE OR REPLACE FUNCTION public.update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Appliquer sur toutes les tables avec updated_at
CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.user_profiles
  FOR EACH ROW EXECUTE FUNCTION public.update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.trips
  FOR EACH ROW EXECUTE FUNCTION public.update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.trip_steps
  FOR EACH ROW EXECUTE FUNCTION public.update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.documents
  FOR EACH ROW EXECUTE FUNCTION public.update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.memories
  FOR EACH ROW EXECUTE FUNCTION public.update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.shared_albums
  FOR EACH ROW EXECUTE FUNCTION public.update_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON public.document_extractions
  FOR EACH ROW EXECUTE FUNCTION public.update_updated_at();
```

---

## 2. ARCHITECTURE DES SERVICES BACKEND

### 2.1 Structure des dossiers — Next.js API Routes

```
src/
├── app/
│   └── api/                              # Next.js API Routes (App Router)
│       ├── auth/
│       │   └── callback/route.ts         # OAuth callback Supabase
│       ├── trips/
│       │   ├── route.ts                  # GET (liste) / POST (créer)
│       │   └── [tripId]/
│       │       ├── route.ts              # GET / PATCH / DELETE
│       │       ├── steps/
│       │       │   ├── route.ts          # GET / POST
│       │       │   └── [stepId]/
│       │       │       ├── route.ts      # GET / PATCH / DELETE
│       │       │       └── reminders/
│       │       │           └── route.ts  # GET / POST / DELETE
│       │       ├── documents/
│       │       │   ├── route.ts          # GET / POST (upload)
│       │       │   └── [documentId]/
│       │       │       └── route.ts      # GET (signed URL) / DELETE
│       │       ├── participants/
│       │       │   ├── route.ts          # GET / POST (inviter)
│       │       │   └── [participantId]/
│       │       │       └── route.ts      # PATCH (rôle) / DELETE
│       │       ├── memories/
│       │       │   ├── route.ts          # GET / POST
│       │       │   └── [memoryId]/
│       │       │       └── route.ts      # GET / PATCH / DELETE
│       │       └── albums/
│       │           ├── route.ts          # GET / POST (créer album)
│       │           └── [albumId]/
│       │               └── route.ts      # GET / PATCH / DELETE
│       ├── invitations/
│       │   └── [token]/
│       │       └── route.ts              # GET (preview) / POST (accepter)
│       ├── tools/
│       │   ├── translate/
│       │   │   └── route.ts              # POST
│       │   ├── currency/
│       │   │   └── route.ts              # GET (taux) / POST (convertir)
│       │   └── timezone/
│       │       └── route.ts              # GET
│       ├── ai/
│       │   ├── suggest-itinerary/
│       │   │   └── route.ts              # POST (Anthropic)
│       │   └── parse-document/
│       │       └── route.ts              # POST (Anthropic)
│       ├── albums/
│       │   └── [slug]/
│       │       └── route.ts              # GET public (album partagé)
│       ├── notifications/
│       │   └── route.ts                  # POST (register token) / GET (historique)
│       └── profile/
│           └── route.ts                  # GET / PATCH
│
├── lib/
│   ├── supabase/
│   │   ├── server.ts                     # Client Supabase côté serveur (cookies)
│   │   ├── admin.ts                      # Client service role (RLS bypassé)
│   │   └── middleware.ts                 # Refresh session middleware
│   ├── services/
│   │   ├── tripService.ts                # Logique métier voyages
│   │   ├── documentService.ts            # Upload, signed URLs, quota
│   │   ├── participantService.ts         # Invitations, tokens, emails
│   │   ├── reminderService.ts            # Calcul et planification rappels
│   │   ├── albumService.ts               # Génération slug, partage
│   │   ├── translateService.ts           # Proxy DeepL + cache
│   │   ├── currencyService.ts            # Proxy Fixer.io + cache
│   │   └── aiService.ts                  # Anthropic client + cache
│   ├── validators/
│   │   ├── trip.schema.ts                # Zod schemas validation
│   │   ├── step.schema.ts
│   │   ├── document.schema.ts
│   │   └── tools.schema.ts
│   └── errors/
│       ├── AppError.ts                   # Classe d'erreur custom
│       └── errorHandler.ts               # Middleware gestion erreurs
│
└── middleware.ts                         # Auth guard global Next.js
```

### 2.2 Modules et responsabilités

| Module | Fichier | Responsabilités | Dépendances |
|---|---|---|---|
| **TripService** | `tripService.ts` | CRUD voyages, calcul statut auto, archivage | Supabase client |
| **DocumentService** | `documentService.ts` | Upload vers Supabase Storage, validation type/taille, génération URLs signées, quota 200Mo/voyage | Supabase Storage, Admin client |
| **ParticipantService** | `participantService.ts` | Génération tokens invitation (crypto), envoi email (Resend), acceptation, gestion rôles | Supabase, Resend API |
| **ReminderService** | `reminderService.ts` | Calcul `remind_at` (step_date - offset, timezone-aware), insertion `step_reminders`, invalidation | Supabase, date-fns-tz |
| **AlbumService** | `albumService.ts` | Génération slug unique (nanoid), hashage password optionnel, gestion expiration, compteur vues | Supabase, bcrypt, nanoid |
| **TranslateService** | `translateService.ts` | Proxy DeepL, hash SHA-256 du texte, lookup cache `translation_cache`, écriture résultat | Supabase, DeepL API |
| **CurrencyService** | `currencyService.ts` | Taux Fixer.io, cache 4h dans `currency_cache`, calcul conversion | Supabase, Fixer.io |
| **AIService** | `aiService.ts` | Client Anthropic, prompt engineering, cache `ai_suggestions` (7j), extraction documents | Anthropic SDK, Supabase |

---

## 3. LISTE COMPLÈTE DES ENDPOINTS API

### 3.1 Authentification & Profil

| Méthode | Route | Auth | Description | Corps / Paramètres |
|---|---|---|---|---|
| `GET` | `/api/auth/callback` | Non | Callback OAuth Supabase | `?code=` (query) |
| `GET` | `/api/profile` | Oui | Récupérer le profil connecté | — |
| `PATCH` | `/api/profile` | Oui | Modifier profil (nom, devise, langue, timezone, push token) | `{ display_name?, default_currency?, push_token? }` |

### 

— BACKEND ENGINEER

### 3.2 Voyages

| Méthode | Route | Auth | Description | Corps / Paramètres |
|---|---|---|---|---|
| `GET` | `/api/trips` | Oui | Lister tous les voyages de l'utilisateur | `?status=upcoming\|ongoing\|past` (query) |
| `POST` | `/api/trips` | Oui | Créer un nouveau voyage | `{ name, destination, country_code, currency_code, timezone, start_date, end_date, cover_image_url? }` |
| `GET` | `/api/trips/:tripId` | Oui | Récupérer le détail d'un voyage | — |
| `PATCH` | `/api/trips/:tripId` | Oui (admin) | Modifier un voyage | `{ name?, destination?, start_date?, end_date?, cover_image_url? }` |
| `DELETE` | `/api/trips/:tripId` | Oui (admin) | Supprimer un voyage (soft delete) | — |

### 3.3 Étapes d'itinéraire

| Méthode | Route | Auth | Description | Corps / Paramètres |
|---|---|---|---|---|
| `GET` | `/api/trips/:tripId/steps` | Oui | Lister les étapes (triées par date/heure) | `?date=YYYY-MM-DD` (filtre journée) |
| `POST` | `/api/trips/:tripId/steps` | Oui (admin\|editor) | Créer une étape | `{ title, step_type, start_at, end_at?, location_name?, location_lat?, location_lng?, notes?, document_ids? }` |
| `GET` | `/api/trips/:tripId/steps/:stepId` | Oui | Détail d'une étape | — |
| `PATCH` | `/api/trips/:tripId/steps/:stepId` | Oui (admin\|editor) | Modifier une étape | Mêmes champs que POST |
| `DELETE` | `/api/trips/:tripId/steps/:stepId` | Oui (admin\|editor) | Supprimer une étape | — |
| `GET` | `/api/trips/:tripId/steps/:stepId/reminders` | Oui | Lister les rappels d'une étape | — |
| `POST` | `/api/trips/:tripId/steps/:stepId/reminders` | Oui | Créer un rappel | `{ offset_minutes, channel }` |
| `DELETE` | `/api/trips/:tripId/steps/:stepId/reminders/:reminderId` | Oui | Supprimer un rappel | — |

### 3.4 Documents

| Méthode | Route | Auth | Description | Corps / Paramètres |
|---|---|---|---|---|
| `GET` | `/api/trips/:tripId/documents` | Oui | Lister les documents d'un voyage | `?category=flight\|accommodation\|activity\|other` |
| `POST` | `/api/trips/:tripId/documents` | Oui (admin\|editor) | Uploader un document | `multipart/form-data: { file, category, step_id? }` |
| `GET` | `/api/trips/:tripId/documents/:documentId` | Oui | Obtenir une URL signée (60 min) | — |
| `DELETE` | `/api/trips/:tripId/documents/:documentId` | Oui (admin\|editor) | Supprimer un document | — |

### 3.5 Participants

| Méthode | Route | Auth | Description | Corps / Paramètres |
|---|---|---|---|---|
| `GET` | `/api/trips/:tripId/participants` | Oui | Lister les participants | — |
| `POST` | `/api/trips/:tripId/participants` | Oui (admin) | Inviter un participant | `{ email, role }` |
| `PATCH` | `/api/trips/:tripId/participants/:participantId` | Oui (admin) | Modifier le rôle | `{ role }` |
| `DELETE` | `/api/trips/:tripId/participants/:participantId` | Oui (admin) | Retirer un participant | — |
| `GET` | `/api/invitations/:token` | Non | Prévisualiser une invitation | — |
| `POST` | `/api/invitations/:token` | Oui | Accepter une invitation | — |

### 3.6 Souvenirs

| Méthode | Route | Auth | Description | Corps / Paramètres |
|---|---|---|---|---|
| `GET` | `/api/trips/:tripId/memories` | Oui | Lister les souvenirs | `?type=photo\|note\|mixed` |
| `POST` | `/api/trips/:tripId/memories` | Oui | Ajouter un souvenir | `multipart/form-data: { file?, caption?, location_name?, location_lat?, location_lng?, taken_at? }` |
| `PATCH` | `/api/trips/:tripId/memories/:memoryId` | Oui (propriétaire) | Modifier un souvenir | `{ caption?, location_name? }` |
| `DELETE` | `/api/trips/:tripId/memories/:memoryId` | Oui (propriétaire\|admin) | Supprimer un souvenir | — |

### 3.7 Albums partagés

| Méthode | Route | Auth | Description | Corps / Paramètres |
|---|---|---|---|---|
| `GET` | `/api/trips/:tripId/albums` | Oui | Lister les albums d'un voyage | — |
| `POST` | `/api/trips/:tripId/albums` | Oui (admin\|editor) | Créer un album partagé | `{ title, memory_ids[], password?, expires_at? }` |
| `PATCH` | `/api/trips/:tripId/albums/:albumId` | Oui (admin) | Modifier un album | `{ title?, memory_ids[]?, expires_at? }` |
| `DELETE` | `/api/trips/:tripId/albums/:albumId` | Oui (admin) | Supprimer un album | — |
| `GET` | `/api/albums/:slug` | Non | Accès public à un album | `?password=` (si protégé) |

### 3.8 Outils utilitaires

| Méthode | Route | Auth | Description | Corps / Paramètres |
|---|---|---|---|---|
| `POST` | `/api/tools/translate` | Oui | Traduire un texte | `{ text, source_lang?, target_lang, trip_id? }` |
| `GET` | `/api/tools/currency` | Oui | Obtenir les taux de change | `?base=EUR&symbols=USD,JPY,GBP` |
| `POST` | `/api/tools/currency` | Oui | Convertir un montant | `{ amount, from_currency, to_currency }` |
| `GET` | `/api/tools/timezone` | Oui | Heure courante d'un fuseau | `?timezone=Asia/Tokyo&compare=Europe/Paris` |

### 3.9 IA (Anthropic)

| Méthode | Route | Auth | Description | Corps / Paramètres |
|---|---|---|---|---|
| `POST` | `/api/ai/suggest-itinerary` | Oui | Suggestions d'itinéraire par IA | `{ trip_id, destination, duration_days, preferences? }` |
| `POST` | `/api/ai/parse-document` | Oui | Extraction données d'un document | `{ document_id }` |

### 3.10 Notifications

| Méthode | Route | Auth | Description | Corps / Paramètres |
|---|---|---|---|---|
| `POST` | `/api/notifications` | Oui | Enregistrer token push device | `{ push_token, platform }` |
| `GET` | `/api/notifications` | Oui | Historique des notifications | `?trip_id=` |

---

## 4. CODE DES ROUTES CLÉS

### 4.1 POST `/api/trips/:tripId/documents` — Upload de document

Cette route est la plus complexe : validation multipart, vérification quota, upload Supabase Storage, extraction IA asynchrone.

```typescript
// src/app/api/trips/[tripId]/documents/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { createServerClient } from '@/lib/supabase/server'
import { createAdminClient } from '@/lib/supabase/admin'
import { documentUploadSchema } from '@/lib/validators/document.schema'
import { AppError } from '@/lib/errors/AppError'

const ALLOWED_MIME_TYPES = ['application/pdf', 'image/jpeg', 'image/png']
const MAX_FILE_SIZE_BYTES = 20 * 1024 * 1024        // 20 Mo par fichier
const MAX_TRIP_STORAGE_BYTES = 200 * 1024 * 1024    // 200 Mo par voyage

export async function GET(
  request: NextRequest,
  { params }: { params: { tripId: string } }
) {
  try {
    const supabase = createServerClient()
    const { data: { user }, error: authError } = await supabase.auth.getUser()
    if (authError || !user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    const { searchParams } = new URL(request.url)
    const category = searchParams.get('category')

    // La RLS vérifie automatiquement l'accès au voyage
    let query = supabase
      .from('documents')
      .select('id, original_name, category, file_size, mime_type, step_id, created_at')
      .eq('trip_id', params.tripId)
      .order('created_at', { ascending: false })

    if (category) {
      query = query.eq('category', category)
    }

    const { data, error } = await query
    if (error) throw new AppError(error.message, 500)

    return NextResponse.json({ documents: data })
  } catch (err) {
    return handleError(err)
  }
}

export async function POST(
  request: NextRequest,
  { params }: { params: { tripId: string } }
) {
  try {
    const supabase = createServerClient()
    const { data: { user }, error: authError } = await supabase.auth.getUser()
    if (authError || !user) {
      return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
    }

    // 1. Vérifier que l'utilisateur est admin ou editor du voyage
    const { data: participant } = await supabase
      .from('trip_participants')
      .select('role')
      .eq('trip_id', params.tripId)
      .eq('user_id', user.id)
      .single()

    if (!participant || !['admin', 'editor'].includes(participant.role)) {
      return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
    }

    // 2. Parser le multipart form
    const formData = await request.formData()
    const file = formData.get('file') as File | null
    const category = formData.get('category') as string
    const stepId = formData.get('step_id') as string | null

    if (!file) {
      return NextResponse.json({ error: 'File is required' }, { status: 400 })
    }

    // 3. Validation type MIME
    if (!ALLOWED_MIME_TYPES.includes(file.type)) {
      return NextResponse.json(
        { error: 'File type not allowed. Accepted: PDF, JPG, PNG' },
        { status: 422 }
      )
    }

    // 4. Validation taille fichier
    if (file.size > MAX_FILE_SIZE_BYTES) {
      return NextResponse.json(
        { error: 'File exceeds 20MB limit' },
        { status: 422 }
      )
    }

    // 5. Validation schema (category, step_id optionnel)
    const validation = documentUploadSchema.safeParse({ category, step_id: stepId })
    if (!validation.success) {
      return NextResponse.json(
        { error: 'Invalid parameters', details: validation.error.flatten() },
        { status: 422 }
      )
    }

    // 6. Vérifier le quota de stockage du voyage (200 Mo total)
    const { data: storageData } = await supabase
      .from('documents')
      .select('file_size')
      .eq('trip_id', params.tripId)

    const currentUsage = (storageData ?? []).reduce(
      (acc, doc) => acc + (doc.file_size ?? 0), 0
    )

    if (currentUsage + file.size > MAX_TRIP_STORAGE_BYTES) {
      return NextResponse.json(
        {
          error: 'Storage quota exceeded',
          current_usage_mb: Math.round(currentUsage / 1024 / 1024),
          limit_mb: 200
        },
        { status: 413 }
      )
    }

    // 7. Upload vers Supabase Storage
    // Chemin : trips/{tripId}/documents/{userId}/{timestamp}_{filename}
    const timestamp = Date.now()
    const safeName = file.name.replace(/[^a-zA-Z0-9._-]/g, '_')
    const storagePath = `trips/${params.tripId}/documents/${user.id}/${timestamp}_${safeName}`

    const arrayBuffer = await file.arrayBuffer()
    const buffer = Buffer.from(arrayBuffer)

    // On utilise le client admin pour bypasser les restrictions Storage RLS
    const adminClient = createAdminClient()
    const { error: uploadError } = await adminClient.storage
      .from('travel-documents')
      .upload(storagePath, buffer, {
        contentType: file.type,
        upsert: false,
      })

    if (uploadError) {
      throw new AppError(`Storage upload failed: ${uploadError.message}`, 500)
    }

    // 8. Insérer en base
    const { data: document, error: dbError } = await supabase
      .from('documents')
      .insert({
        trip_id: params.tripId,
        step_id: stepId || null,
        uploaded_by: user.id,
        original_name: file.name,
        storage_path: storagePath,
        mime_type: file.type,
        file_size: file.size,
        category: validation.data.category,
      })
      .select('id, original_name, category, file_size, created_at')
      .single()

    if (dbError) {
      // Rollback du fichier uploadé si l'insertion échoue
      await adminClient.storage
        .from('travel-documents')
        .remove([storagePath])
      throw new AppError(dbError.message, 500)
    }

    // 9. Déclencher l'extraction IA en arrière-plan (non-bloquant)
    // On ne await pas — la réponse est retournée immédiatement
    triggerDocumentExtraction(document.id, params.tripId, storagePath, file.type)
      .catch(err => console.error('[AI Extraction] Failed:', err))

    return NextResponse.json({ document }, { status: 201 })
  } catch (err) {
    return handleError(err)
  }
}

/**
 * Déclenche l'extraction IA asynchrone d'un document.
 * Appel non-bloquant : la route POST renvoie 201 avant la fin de l'extraction.
 */
async function triggerDocumentExtraction(
  documentId: string,
  tripId: string,
  storagePath: string,

— BACKEND ENGINEER

# Livrable Backend Engineer — Travel Companion App
## MVP B2C · Next.js API Routes · Supabase · Anthropic

---

## 1. SCHÉMA DE BASE DE DONNÉES

### 1.1 Vue d'ensemble des tables

```
users                     (géré par Supabase Auth + extension profil)
trips                     (voyages)
trip_participants         (membres d'un voyage avec rôles)
trip_invitations          (invitations en attente)
itinerary_steps           (étapes d'un itinéraire)
documents                 (fichiers importés)
memories                  (souvenirs photo/note/lieu)
shared_albums             (albums partagés publiquement)
shared_album_memories     (pivot album ↔ souvenirs)
device_tokens             (tokens push FCM/APNs)
notification_preferences  (préférences notifications par voyage)
scheduled_notifications   (rappels planifiés)
currency_rates_cache      (cache des taux de change)
```

---

### 1.2 Schéma SQL complet

```sql
-- ============================================================
-- EXTENSION UUID
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- ============================================================
-- TABLE : profiles
-- Extension de auth.users Supabase
-- ============================================================
CREATE TABLE profiles (
  id            UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name  TEXT,
  avatar_url    TEXT,
  default_currency  CHAR(3) DEFAULT 'EUR',
  default_language  CHAR(5) DEFAULT 'fr',
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- TABLE : trips
-- ============================================================
CREATE TABLE trips (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  owner_id      UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  name          TEXT NOT NULL CHECK (char_length(name) BETWEEN 1 AND 100),
  description   TEXT CHECK (char_length(description) <= 500),
  cover_image_url TEXT,
  
  -- Destination principale (peut être multi-destinations via itinerary_steps)
  primary_destination TEXT NOT NULL,
  primary_country_code CHAR(2),           -- ISO 3166-1 alpha-2
  primary_timezone    TEXT,               -- ex: "Asia/Tokyo"
  primary_currency    CHAR(3),            -- ISO 4217 ex: "JPY"
  primary_language    CHAR(5),            -- ex: "ja"
  
  start_date    DATE NOT NULL,
  end_date      DATE NOT NULL CHECK (end_date >= start_date),
  
  status        TEXT NOT NULL DEFAULT 'upcoming'
                CHECK (status IN ('upcoming', 'ongoing', 'past', 'archived')),
  
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  CONSTRAINT valid_dates CHECK (end_date >= start_date)
);

CREATE INDEX idx_trips_owner_id ON trips(owner_id);
CREATE INDEX idx_trips_status ON trips(status);

-- ============================================================
-- TABLE : trip_participants
-- ============================================================
CREATE TABLE trip_participants (
  id        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id   UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  user_id   UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  role      TEXT NOT NULL DEFAULT 'viewer'
            CHECK (role IN ('admin', 'editor', 'viewer')),
  joined_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  UNIQUE(trip_id, user_id)
);

CREATE INDEX idx_trip_participants_trip_id ON trip_participants(trip_id);
CREATE INDEX idx_trip_participants_user_id ON trip_participants(user_id);

-- ============================================================
-- TABLE : trip_invitations
-- ============================================================
CREATE TABLE trip_invitations (
  id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id     UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  invited_by  UUID NOT NULL REFERENCES auth.users(id),
  email       TEXT NOT NULL,
  role        TEXT NOT NULL DEFAULT 'viewer'
              CHECK (role IN ('editor', 'viewer')),
  token       TEXT NOT NULL UNIQUE DEFAULT encode(gen_random_bytes(32), 'hex'),
  status      TEXT NOT NULL DEFAULT 'pending'
              CHECK (status IN ('pending', 'accepted', 'expired', 'cancelled')),
  expires_at  TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '7 days',
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invitations_trip_id ON trip_invitations(trip_id);
CREATE INDEX idx_invitations_token ON trip_invitations(token);
CREATE INDEX idx_invitations_email ON trip_invitations(email);

-- ============================================================
-- TABLE : itinerary_steps
-- ============================================================
CREATE TABLE itinerary_steps (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id       UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  
  title         TEXT NOT NULL CHECK (char_length(title) BETWEEN 1 AND 200),
  description   TEXT CHECK (char_length(description) <= 2000),
  category      TEXT NOT NULL DEFAULT 'other'
                CHECK (category IN ('flight', 'accommodation', 'activity', 'transport', 'restaurant', 'other')),
  
  -- Localisation
  location_name TEXT,
  location_address TEXT,
  latitude      DECIMAL(10, 8),
  longitude     DECIMAL(11, 8),
  
  -- Temporalité
  step_date     DATE NOT NULL,
  start_time    TIME,
  end_time      TIME,
  timezone      TEXT,                   -- Fuseau propre à l'étape si différent
  
  -- Métadonnées
  confirmation_number TEXT,
  notes         TEXT CHECK (char_length(notes) <= 2000),
  position      INTEGER NOT NULL DEFAULT 0,   -- Ordre dans la journée
  
  -- IA extraction
  ai_extracted  BOOLEAN DEFAULT false,
  
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_steps_trip_id ON itinerary_steps(trip_id);
CREATE INDEX idx_steps_step_date ON itinerary_steps(step_date);
CREATE INDEX idx_steps_trip_date ON itinerary_steps(trip_id, step_date);

-- ============================================================
-- TABLE : documents
-- ============================================================
CREATE TABLE documents (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id       UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  step_id       UUID REFERENCES itinerary_steps(id) ON DELETE SET NULL,
  uploaded_by   UUID NOT NULL REFERENCES auth.users(id),
  
  original_name TEXT NOT NULL,
  storage_path  TEXT NOT NULL UNIQUE,   -- Chemin Supabase Storage
  mime_type     TEXT NOT NULL
                CHECK (mime_type IN ('application/pdf', 'image/jpeg', 'image/png')),
  file_size     INTEGER NOT NULL CHECK (file_size > 0 AND file_size <= 20971520), -- 20 Mo max
  
  category      TEXT NOT NULL DEFAULT 'other'
                CHECK (category IN ('flight', 'accommodation', 'activity', 'other')),
  
  -- Extraction IA
  ai_status     TEXT NOT NULL DEFAULT 'pending'
                CHECK (ai_status IN ('pending', 'processing', 'completed', 'failed', 'skipped')),
  ai_extracted_data JSONB,              -- Données structurées extraites
  
  -- Cache URL signée
  signed_url    TEXT,
  signed_url_expires_at TIMESTAMPTZ,
  
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_documents_trip_id ON documents(trip_id);
CREATE INDEX idx_documents_step_id ON documents(step_id);
CREATE INDEX idx_documents_category ON documents(trip_id, category);
CREATE INDEX idx_documents_ai_status ON documents(ai_status) WHERE ai_status = 'pending';

-- ============================================================
-- TABLE : memories
-- ============================================================
CREATE TABLE memories (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id       UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  step_id       UUID REFERENCES itinerary_steps(id) ON DELETE SET NULL,
  author_id     UUID NOT NULL REFERENCES auth.users(id),
  
  type          TEXT NOT NULL CHECK (type IN ('photo', 'note', 'photo_note')),
  caption       TEXT CHECK (char_length(caption) <= 500),
  
  -- Photo
  storage_path  TEXT,                   -- Chemin Supabase Storage (si photo)
  thumbnail_path TEXT,                  -- Version compressée
  
  -- Localisation optionnelle
  location_name TEXT,
  latitude      DECIMAL(10, 8),
  longitude     DECIMAL(11, 8),
  
  -- Métadonnées
  taken_at      TIMESTAMPTZ,            -- Date prise de vue (EXIF ou manuelle)
  position      INTEGER DEFAULT 0,
  
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_memories_trip_id ON memories(trip_id);
CREATE INDEX idx_memories_author_id ON memories(author_id);

-- ============================================================
-- TABLE : shared_albums
-- ============================================================
CREATE TABLE shared_albums (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id       UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  created_by    UUID NOT NULL REFERENCES auth.users(id),
  
  title         TEXT NOT NULL CHECK (char_length(title) BETWEEN 1 AND 100),
  slug          TEXT NOT NULL UNIQUE,   -- URL publique : /share/{slug}
  
  -- Protection optionnelle
  password_hash TEXT,                   -- bcrypt hash si album protégé
  
  -- Expiration optionnelle
  expires_at    TIMESTAMPTZ,
  
  -- Stats
  view_count    INTEGER NOT NULL DEFAULT 0,
  
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_albums_trip_id ON shared_albums(trip_id);
CREATE UNIQUE INDEX idx_albums_slug ON shared_albums(slug);

-- ============================================================
-- TABLE : shared_album_memories (pivot)
-- ============================================================
CREATE TABLE shared_album_memories (
  album_id    UUID NOT NULL REFERENCES shared_albums(id) ON DELETE CASCADE,
  memory_id   UUID NOT NULL REFERENCES memories(id) ON DELETE CASCADE,
  position    INTEGER NOT NULL DEFAULT 0,
  added_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  PRIMARY KEY (album_id, memory_id)
);

-- ============================================================
-- TABLE : device_tokens
-- ============================================================
CREATE TABLE device_tokens (
  id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id     UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  token       TEXT NOT NULL,
  platform    TEXT NOT NULL CHECK (platform IN ('ios', 'android', 'web')),
  is_active   BOOLEAN NOT NULL DEFAULT true,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  UNIQUE(user_id, token)
);

CREATE INDEX idx_device_tokens_user_id ON device_tokens(user_id) WHERE is_active = true;

-- ============================================================
-- TABLE : notification_preferences
-- ============================================================
CREATE TABLE notification_preferences (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  trip_id         UUID REFERENCES trips(id) ON DELETE CASCADE,    -- NULL = préférence globale
  
  enabled         BOOLEAN NOT NULL DEFAULT true,
  reminder_24h    BOOLEAN NOT NULL DEFAULT true,
  reminder_2h     BOOLEAN NOT NULL DEFAULT true,
  reminder_custom_minutes INTEGER,                -- NULL = désactivé
  
  UNIQUE(user_id, trip_id)
);

-- ============================================================
-- TABLE : scheduled_notifications
-- ============================================================
CREATE TABLE scheduled_notifications (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  step_id       UUID NOT NULL REFERENCES itinerary_steps(id) ON DELETE CASCADE,
  user_id       UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  
  type          TEXT NOT NULL CHECK (type IN ('reminder_24h', 'reminder_2h', 'reminder_custom')),
  scheduled_for TIMESTAMPTZ NOT NULL,
  sent_at       TIMESTAMPTZ,
  status        TEXT NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending', 'sent', 'failed', 'cancelled')),
  
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  
  UNIQUE(step_id, user_id, type)
);

CREATE INDEX idx_scheduled_notif_pending ON scheduled_notifications(scheduled_for)
  WHERE status = 'pending';

-- ============================================================
-- TABLE : currency_rates_cache
-- ============================================================
CREATE TABLE currency_rates_cache (
  base_currency CHAR(3) NOT NULL,
  rates         JSONB NOT NULL,
  fetched_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at    TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '6 hours',
  
  PRIMARY KEY (base_currency)
);

-- ============================================================
-- TRIGGERS : updated_at automatique
-- ============================================================
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_profiles_updated_at
  BEFORE UPDATE ON profiles FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER trg_trips_updated_at
  BEFORE UPDATE ON trips FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER trg_steps_updated_at
  BEFORE UPDATE ON itinerary_steps FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER trg_documents_updated_at
  BEFORE UPDATE ON documents FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER trg_memories_updated_at
  BEFORE UPDATE ON memories FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER trg_albums_updated_at
  BEFORE UPDATE ON shared_albums FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- ============================================================
-- TRIGGER : statut automatique des voyages
-- ============================================================
CREATE OR REPLACE FUNCTION auto_update_trip_status()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.start_date <= CURRENT_DATE AND NEW.end_date >= CURRENT_DATE THEN
    NEW.status = 'ongoing';
  ELSIF NEW.end_date < CURRENT_DATE THEN
    NEW.status = 'past';
  ELSE
    NEW.status = 'upcoming';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_trip_status
  BEFORE INSERT OR UPDATE OF start_date, end_date ON trips
  FOR EACH ROW EXECUTE FUNCTION auto_update_trip_status();

-- ============================================================
-- TRIGGER : insertion auto du créateur comme admin
-- ============================================================
CREATE OR REPLACE FUNCTION add_trip_owner_as_admin()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO trip_participants (trip_id, user_id, role)
  VALUES (NEW.id, NEW.owner_id, 'admin');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_trip_owner_participant

— BACKEND ENGINEER

# Livrable Backend Engineer — Travel Companion App
## MVP B2C · Next.js API Routes · Supabase · Anthropic API

---

## 1. SCHÉMA DE BASE DE DONNÉES COMPLET

> Le schéma SQL a été largement produit dans l'itération précédente. Cette section complète les éléments manquants, corrige les gaps et documente les décisions de conception.

### 1.1 Rappel des tables existantes (inventaire complet)

| Table | Rôle |
|---|---|
| `profiles` | Extension de `auth.users` Supabase — données métier utilisateur |
| `trips` | Voyage — entité centrale |
| `trip_participants` | Pivot utilisateur ↔ voyage avec rôle |
| `itinerary_steps` | Étapes de l'itinéraire |
| `documents` | Fichiers importés associés à un voyage/étape |
| `memories` | Souvenirs (photo, note, photo+note) |
| `shared_albums` | Albums partagés publiquement |
| `shared_album_memories` | Pivot album ↔ souvenir |
| `device_tokens` | Tokens push (FCM/APNs) |
| `notification_preferences` | Préférences notifications par user/voyage |
| `scheduled_notifications` | File de notifications planifiées |
| `currency_rates_cache` | Cache taux de change |

### 1.2 Tables manquantes à ajouter

```sql
-- ============================================================
-- TABLE : trip_invitations
-- Gestion des invitations par lien ou email
-- ============================================================
CREATE TABLE trip_invitations (
  id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id     UUID NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  invited_by  UUID NOT NULL REFERENCES auth.users(id),
  
  -- Invitation par email OU par lien token
  email       TEXT,                           -- NULL si invitation par lien générique
  token       TEXT NOT NULL UNIQUE,           -- Token URL sécurisé (32 bytes hex)
  role        TEXT NOT NULL DEFAULT 'viewer'
              CHECK (role IN ('viewer', 'editor', 'admin')),
  
  status      TEXT NOT NULL DEFAULT 'pending'
              CHECK (status IN ('pending', 'accepted', 'declined', 'expired')),
  
  expires_at  TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '7 days',
  accepted_at TIMESTAMPTZ,
  
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invitations_token ON trip_invitations(token) WHERE status = 'pending';
CREATE INDEX idx_invitations_email ON trip_invitations(email) WHERE email IS NOT NULL;
CREATE INDEX idx_invitations_trip_id ON trip_invitations(trip_id);

-- ============================================================
-- TABLE : user_preferences
-- Préférences globales utilisateur (devise, langue, fuseau)
-- ============================================================
CREATE TABLE user_preferences (
  user_id           UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  
  default_currency  CHAR(3) NOT NULL DEFAULT 'EUR',
  default_language  CHAR(5) NOT NULL DEFAULT 'fr',   -- BCP 47 : 'fr', 'en', 'es'...
  timezone          TEXT NOT NULL DEFAULT 'Europe/Paris',
  
  -- Notification globales
  push_enabled      BOOLEAN NOT NULL DEFAULT true,
  
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- TABLE : translation_cache
-- Cache des traductions pour limiter les appels API DeepL
-- ============================================================
CREATE TABLE translation_cache (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  source_text   TEXT NOT NULL,
  source_lang   CHAR(5) NOT NULL,
  target_lang   CHAR(5) NOT NULL,
  translated    TEXT NOT NULL,
  cached_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  hit_count     INTEGER NOT NULL DEFAULT 1,
  
  -- Clé de déduplication : hash du texte source + langues
  content_hash  TEXT NOT NULL,
  UNIQUE(content_hash)
);

CREATE INDEX idx_translation_cache_hash ON translation_cache(content_hash);
-- Purge automatique après 30 jours (géré par cron ou pg_cron)
CREATE INDEX idx_translation_cache_age ON translation_cache(cached_at);
```

### 1.3 Row Level Security (RLS) — Politique complète

> Supabase impose une stratégie RLS par table. Voici les politiques critiques.

```sql
-- ============================================================
-- ACTIVATION RLS sur toutes les tables
-- ============================================================
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE trips ENABLE ROW LEVEL SECURITY;
ALTER TABLE trip_participants ENABLE ROW LEVEL SECURITY;
ALTER TABLE trip_invitations ENABLE ROW LEVEL SECURITY;
ALTER TABLE itinerary_steps ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE memories ENABLE ROW LEVEL SECURITY;
ALTER TABLE shared_albums ENABLE ROW LEVEL SECURITY;
ALTER TABLE shared_album_memories ENABLE ROW LEVEL SECURITY;
ALTER TABLE device_tokens ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_preferences ENABLE ROW LEVEL SECURITY;
ALTER TABLE scheduled_notifications ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_preferences ENABLE ROW LEVEL SECURITY;
-- NB: currency_rates_cache et translation_cache : accès via service_role uniquement

-- ============================================================
-- HELPER FUNCTION : vérifier si l'utilisateur est participant
-- ============================================================
CREATE OR REPLACE FUNCTION is_trip_participant(trip_uuid UUID)
RETURNS BOOLEAN AS $$
  SELECT EXISTS (
    SELECT 1 FROM trip_participants
    WHERE trip_id = trip_uuid
    AND user_id = auth.uid()
  );
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Vérifier le rôle minimum sur un voyage
CREATE OR REPLACE FUNCTION has_trip_role(trip_uuid UUID, min_role TEXT)
RETURNS BOOLEAN AS $$
  SELECT EXISTS (
    SELECT 1 FROM trip_participants
    WHERE trip_id = trip_uuid
    AND user_id = auth.uid()
    AND CASE min_role
      WHEN 'viewer' THEN role IN ('viewer', 'editor', 'admin')
      WHEN 'editor' THEN role IN ('editor', 'admin')
      WHEN 'admin'  THEN role = 'admin'
      ELSE false
    END
  );
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- ============================================================
-- POLICIES : profiles
-- ============================================================
CREATE POLICY "Users can view their own profile"
  ON profiles FOR SELECT USING (id = auth.uid());

CREATE POLICY "Users can update their own profile"
  ON profiles FOR UPDATE USING (id = auth.uid());

-- Voir le profil des co-participants (pour affichage noms)
CREATE POLICY "Users can view co-participants profiles"
  ON profiles FOR SELECT USING (
    EXISTS (
      SELECT 1 FROM trip_participants tp1
      JOIN trip_participants tp2 ON tp1.trip_id = tp2.trip_id
      WHERE tp1.user_id = auth.uid()
      AND tp2.user_id = profiles.id
    )
  );

-- ============================================================
-- POLICIES : trips
-- ============================================================
CREATE POLICY "Participants can view their trips"
  ON trips FOR SELECT USING (is_trip_participant(id));

CREATE POLICY "Authenticated users can create trips"
  ON trips FOR INSERT WITH CHECK (auth.uid() = owner_id);

CREATE POLICY "Admins can update their trips"
  ON trips FOR UPDATE USING (has_trip_role(id, 'admin'));

CREATE POLICY "Admins can delete their trips"
  ON trips FOR DELETE USING (has_trip_role(id, 'admin'));

-- ============================================================
-- POLICIES : itinerary_steps
-- ============================================================
CREATE POLICY "Participants can view steps"
  ON itinerary_steps FOR SELECT USING (is_trip_participant(trip_id));

CREATE POLICY "Editors can insert steps"
  ON itinerary_steps FOR INSERT WITH CHECK (has_trip_role(trip_id, 'editor'));

CREATE POLICY "Editors can update steps"
  ON itinerary_steps FOR UPDATE USING (has_trip_role(trip_id, 'editor'));

CREATE POLICY "Admins can delete steps"
  ON itinerary_steps FOR DELETE USING (has_trip_role(trip_id, 'admin'));

-- ============================================================
-- POLICIES : documents
-- ============================================================
CREATE POLICY "Participants can view documents"
  ON documents FOR SELECT USING (is_trip_participant(trip_id));

CREATE POLICY "Editors can upload documents"
  ON documents FOR INSERT WITH CHECK (
    has_trip_role(trip_id, 'editor')
    AND uploader_id = auth.uid()
  );

CREATE POLICY "Uploader or admin can delete document"
  ON documents FOR DELETE USING (
    uploader_id = auth.uid() OR has_trip_role(trip_id, 'admin')
  );

-- ============================================================
-- POLICIES : memories
-- ============================================================
CREATE POLICY "Participants can view memories"
  ON memories FOR SELECT USING (is_trip_participant(trip_id));

CREATE POLICY "Participants can add memories"
  ON memories FOR INSERT WITH CHECK (
    is_trip_participant(trip_id)
    AND author_id = auth.uid()
  );

CREATE POLICY "Author or admin can delete memory"
  ON memories FOR DELETE USING (
    author_id = auth.uid() OR has_trip_role(trip_id, 'admin')
  );

-- ============================================================
-- POLICIES : shared_albums (accès public en lecture)
-- ============================================================
-- L'accès public est géré au niveau API (anon key + vérification slug)
-- La politique RLS protège les opérations authentifiées

CREATE POLICY "Album creator can manage album"
  ON shared_albums FOR ALL USING (created_by = auth.uid());

CREATE POLICY "Trip participants can view albums"
  ON shared_albums FOR SELECT USING (is_trip_participant(trip_id));

-- ============================================================
-- POLICIES : device_tokens
-- ============================================================
CREATE POLICY "Users manage their own device tokens"
  ON device_tokens FOR ALL USING (user_id = auth.uid());

-- ============================================================
-- POLICIES : user_preferences
-- ============================================================
CREATE POLICY "Users manage their own preferences"
  ON user_preferences FOR ALL USING (user_id = auth.uid());
```

### 1.4 Storage Buckets Supabase

```sql
-- Buckets à créer dans Supabase Storage
-- NB : configuration via dashboard ou API Supabase Management

-- Bucket "documents" : fichiers de voyage (PDF, images)
-- Accès : privé, URLs signées uniquement
-- Quota : 20 Mo par fichier, 500 Mo par voyage (à enforcer côté API)

-- Bucket "memories" : photos souvenirs
-- Accès : privé par défaut, public si dans un album partagé
-- Quota : 10 Mo par photo (resize côté upload)

-- Bucket "thumbnails" : miniatures auto-générées
-- Accès : public (optimisation performance)
-- Généré par : Edge Function Supabase trigger sur upload memories

-- Politique Storage RLS — documents
-- (configurée dans Supabase Dashboard)
-- SELECT : is_trip_participant(trip_id extrait du path)
-- INSERT : has_trip_role(trip_id, 'editor')
-- DELETE : uploader = auth.uid() OR has_trip_role(trip_id, 'admin')

-- Convention de nommage des paths :
-- documents : {trip_id}/{document_id}/{filename}
-- memories  : {trip_id}/{memory_id}/{filename}
-- thumbnails: {trip_id}/{memory_id}/thumb_{filename}
```

---

## 2. ARCHITECTURE DES SERVICES ET MODULES

### 2.1 Organisation des API Routes (Next.js)

```
src/
├── app/
│   └── api/                              # API Routes Next.js
│       ├── auth/
│       │   ├── register/route.ts
│       │   ├── login/route.ts            # Email/password (Supabase Auth)
│       │   ├── logout/route.ts
│       │   ├── refresh/route.ts
│       │   └── me/route.ts
│       │
│       ├── trips/
│       │   ├── route.ts                  # GET (list) / POST (create)
│       │   └── [tripId]/
│       │       ├── route.ts              # GET / PATCH / DELETE
│       │       ├── itinerary/
│       │       │   ├── route.ts          # GET (full itinerary)
│       │       │   └── steps/
│       │       │       ├── route.ts      # POST (create step)
│       │       │       └── [stepId]/
│       │       │           └── route.ts  # GET / PATCH / DELETE
│       │       ├── documents/
│       │       │   ├── route.ts          # GET (list) / POST (init upload)
│       │       │   └── [documentId]/
│       │       │       ├── route.ts      # GET / DELETE
│       │       │       └── signed-url/route.ts  # GET (URL signée lecture)
│       │       ├── participants/
│       │       │   ├── route.ts          # GET (list)
│       │       │   └── [userId]/route.ts # PATCH (role) / DELETE (remove)
│       │       ├── invitations/
│       │       │   ├── route.ts          # POST (create invitation)
│       │       │   └── [invitationId]/route.ts  # DELETE (revoke)
│       │       ├── memories/
│       │       │   ├── route.ts          # GET (list) / POST (create)
│       │       │   └── [memoryId]/route.ts      # PATCH / DELETE
│       │       └── albums/
│       │           ├── route.ts          # GET (list) / POST (create)
│       │           └── [albumId]/
│       │               ├── route.ts      # PATCH / DELETE
│       │               └── memories/route.ts    # POST/DELETE (add/remove)
│       │
│       ├── invitations/
│       │   └── [token]/
│       │       └── route.ts              # GET (preview) / POST (accept)
│       │
│       ├── share/
│       │   └── [slug]/route.ts           # GET public (album public)
│       │
│       ├── tools/
│       │   ├── translate/route.ts        # POST
│       │   ├── currency/
│       │   │   ├── rates/route.ts        # GET (taux pour une base)
│       │   │   └── convert/route.ts      # POST (conversion)
│       │   └── timezone/route.ts         # GET (info fuseau destination)
│       │
│       ├── notifications/
│       │   ├── device-token/route.ts     # POST (enregistrer) / DELETE (supprimer)
│       │   └── preferences/
│       │       ├── route.ts              # GET/PUT (préférences globales)
│       │       └── [tripId]/route.ts     # GET/PUT (préférences par voyage)
│       │
│       └── users/
│           └── preferences/route.ts      # GET / PUT
│
├── lib/
│   ├── supabase/
│   │   ├── server.ts                     # Client Supabase côté serveur (service_role)
│   │   ├── client.ts                     # Client Supabase côté client (anon)
│   │   └── middleware.ts                 # Middleware session refresh
│   │
│   ├── services/                         

— BACKEND ENGINEER

# Livrable Backend Engineer — Travel Companion App
## MVP B2C · Next.js API Routes · Supabase · Anthropic (si applicable)

---

> **Reprise à partir de `lib/services/`** — Ce livrable continue directement là où le précédent s'est interrompu. La section 1 (Schéma BDD + RLS + Storage) et le début de la section 2 (arborescence API Routes) ont déjà été produits. Ce document couvre la suite complète.

---

## 2. ARCHITECTURE DES SERVICES ET MODULES (suite)

### 2.2 Couche Services — `lib/services/`

```
lib/
└── services/
    ├── tripService.ts          # Logique métier voyages, participants, statuts
    ├── itineraryService.ts     # CRUD étapes, calcul vue journalière
    ├── documentService.ts      # Upload orchestration, URLs signées, quotas
    ├── memoryService.ts        # Gestion souvenirs, albums, liens publics
    ├── toolsService.ts         # Proxy APIs tierces, cache taux de change
    ├── notificationService.ts  # Planification rappels, envoi push
    ├── invitationService.ts    # Génération tokens, validation, expiration
    └── storageService.ts       # Abstraction Supabase Storage (upload, delete, sign)
```

**Principe d'organisation** : chaque route API ne contient que la validation de la requête et la sérialisation de la réponse. Toute logique métier est dans les services. Les services appellent directement le client Supabase server-side.

### 2.3 Couche Utilitaires — `lib/utils/`

```
lib/
└── utils/
    ├── auth.ts             # Extraction session depuis cookies Supabase
    ├── errors.ts           # Classes d'erreurs typées, handler centralisé
    ├── validation.ts       # Schemas Zod partagés (trip, step, document...)
    ├── pagination.ts       # Helpers curseur/offset pour listes paginées
    ├── slugify.ts          # Génération slugs uniques pour albums partagés
    └── rateLimit.ts        # Wrapper rate limiting par IP et par user
```

### 2.4 Client Supabase — Dual Mode

```typescript
// lib/supabase/server.ts
// Client avec service_role pour les opérations backend sensibles
// NE JAMAIS exposer ce client côté client/browser

import { createClient } from '@supabase/supabase-js'
import type { Database } from '@/types/supabase'

// Client standard : respecte les RLS policies
export function createServerClient() {
  return createClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}

// Client admin : bypass RLS — réservé aux opérations système
// (planification notifications, génération thumbnails, cleanup)
export function createAdminClient() {
  return createClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!  // Jamais exposé côté client
  )
}
```

---

## 3. LISTE COMPLÈTE DES ENDPOINTS API

### Convention générale

| Élément | Convention |
|---|---|
| Base URL | `/api/` (Next.js App Router) |
| Versioning | Pas de préfixe `/v1/` en MVP — ajouté lors du passage B2B |
| Auth | Cookie `sb-access-token` (Supabase Auth) ou header `Authorization: Bearer <token>` |
| Format | JSON systématique, `Content-Type: application/json` |
| Erreurs | Objet `{ error: { code: string, message: string, details?: any } }` |
| Pagination | Query params `?page=1&limit=20` ou curseur `?cursor=<id>` selon le cas |
| Timestamps | ISO 8601 UTC |

---

### 3.1 Module AUTH

| Méthode | Endpoint | Auth requise | Description |
|---|---|---|---|
| POST | `/api/auth/register` | Non | Inscription email/password |
| POST | `/api/auth/login` | Non | Connexion email/password |
| POST | `/api/auth/logout` | Oui | Déconnexion + invalidation session |
| POST | `/api/auth/refresh` | Non (cookie) | Rafraîchissement token silencieux |
| GET | `/api/auth/me` | Oui | Profil utilisateur courant |
| POST | `/api/auth/oauth/google` | Non | Initiation flow OAuth Google |
| POST | `/api/auth/oauth/apple` | Non | Initiation flow OAuth Apple |

**Note** : En Supabase, register/login/OAuth sont directement gérés par le SDK client Supabase Auth. Les routes `/api/auth/` servent de proxy server-side pour sécuriser les cookies et centraliser la logique post-auth (création profil, préférences par défaut).

---

### 3.2 Module TRIPS

| Méthode | Endpoint | Auth | Description | Params clés |
|---|---|---|---|---|
| GET | `/api/trips` | Oui | Liste des voyages de l'utilisateur | `?status=upcoming\|ongoing\|past` |
| POST | `/api/trips` | Oui | Création d'un nouveau voyage | Body: `TripCreatePayload` |
| GET | `/api/trips/[tripId]` | Oui | Détail d'un voyage | — |
| PATCH | `/api/trips/[tripId]` | Oui (admin) | Modification d'un voyage | Body: `TripUpdatePayload` |
| DELETE | `/api/trips/[tripId]` | Oui (admin) | Suppression d'un voyage | — |
| GET | `/api/trips/[tripId]/itinerary` | Oui | Itinéraire complet (toutes étapes groupées par jour) | `?timezone=Europe/Paris` |
| GET | `/api/trips/[tripId]/itinerary/steps` | Oui | Liste plate des étapes | `?date=2024-07-15` |
| POST | `/api/trips/[tripId]/itinerary/steps` | Oui (editor) | Ajout d'une étape | Body: `StepCreatePayload` |
| GET | `/api/trips/[tripId]/itinerary/steps/[stepId]` | Oui | Détail d'une étape | — |
| PATCH | `/api/trips/[tripId]/itinerary/steps/[stepId]` | Oui (editor) | Modification d'une étape | Body: `StepUpdatePayload` |
| DELETE | `/api/trips/[tripId]/itinerary/steps/[stepId]` | Oui (admin) | Suppression d'une étape | — |

---

### 3.3 Module PARTICIPANTS & INVITATIONS

| Méthode | Endpoint | Auth | Description | Params clés |
|---|---|---|---|---|
| GET | `/api/trips/[tripId]/participants` | Oui | Liste des participants du voyage | — |
| PATCH | `/api/trips/[tripId]/participants/[userId]` | Oui (admin) | Changement de rôle d'un participant | Body: `{ role: 'admin'\|'editor'\|'viewer' }` |
| DELETE | `/api/trips/[tripId]/participants/[userId]` | Oui (admin) | Retrait d'un participant | — |
| POST | `/api/trips/[tripId]/invitations` | Oui (admin) | Création d'une invitation | Body: `{ email?: string, role: string, expires_at?: date }` |
| GET | `/api/trips/[tripId]/invitations` | Oui (admin) | Liste des invitations actives | — |
| DELETE | `/api/trips/[tripId]/invitations/[invitationId]` | Oui (admin) | Révocation d'une invitation | — |
| GET | `/api/invitations/[token]` | Non | Aperçu d'une invitation (avant acceptation) | — |
| POST | `/api/invitations/[token]/accept` | Oui | Acceptation d'une invitation | — |

---

### 3.4 Module DOCUMENTS

| Méthode | Endpoint | Auth | Description | Params clés |
|---|---|---|---|---|
| GET | `/api/trips/[tripId]/documents` | Oui | Liste documents du voyage | `?category=flight\|accommodation\|activity\|other` |
| POST | `/api/trips/[tripId]/documents` | Oui (editor) | Initialisation d'un upload (retourne URL présignée) | Body: `DocumentInitPayload` |
| POST | `/api/trips/[tripId]/documents/confirm` | Oui (editor) | Confirmation upload terminé (metadata) | Body: `DocumentConfirmPayload` |
| GET | `/api/trips/[tripId]/documents/[documentId]` | Oui | Métadonnées d'un document | — |
| GET | `/api/trips/[tripId]/documents/[documentId]/signed-url` | Oui | URL signée de lecture (expiration 1h) | — |
| DELETE | `/api/trips/[tripId]/documents/[documentId]` | Oui | Suppression document (uploader ou admin) | — |

**Flux upload en deux temps** :
1. `POST /documents` → le backend valide quota + type, retourne une URL présignée Supabase Storage
2. Le client uploade directement vers Supabase Storage
3. `POST /documents/confirm` → le backend enregistre les métadonnées en base

Ce pattern évite de faire transiter les fichiers binaires par Next.js.

---

### 3.5 Module MEMORIES & ALBUMS

| Méthode | Endpoint | Auth | Description | Params clés |
|---|---|---|---|---|
| GET | `/api/trips/[tripId]/memories` | Oui | Liste des souvenirs du voyage | `?page=1&limit=20` |
| POST | `/api/trips/[tripId]/memories` | Oui | Ajout d'un souvenir | Body: `MemoryCreatePayload` |
| PATCH | `/api/trips/[tripId]/memories/[memoryId]` | Oui | Modification (auteur ou admin) | Body: `MemoryUpdatePayload` |
| DELETE | `/api/trips/[tripId]/memories/[memoryId]` | Oui | Suppression (auteur ou admin) | — |
| GET | `/api/trips/[tripId]/albums` | Oui | Liste des albums du voyage | — |
| POST | `/api/trips/[tripId]/albums` | Oui (admin) | Création d'un album partagé | Body: `AlbumCreatePayload` |
| PATCH | `/api/trips/[tripId]/albums/[albumId]` | Oui (admin) | Modification album (titre, visibilité) | — |
| DELETE | `/api/trips/[tripId]/albums/[albumId]` | Oui (admin) | Suppression album + révocation lien | — |
| POST | `/api/trips/[tripId]/albums/[albumId]/memories` | Oui (admin) | Ajout de souvenirs à l'album | Body: `{ memory_ids: string[] }` |
| DELETE | `/api/trips/[tripId]/albums/[albumId]/memories/[memoryId]` | Oui (admin) | Retrait d'un souvenir de l'album | — |
| GET | `/api/share/[slug]` | Non | Vue publique d'un album (accès invité) | — |

---

### 3.6 Module TOOLS

| Méthode | Endpoint | Auth | Description | Params clés |
|---|---|---|---|---|
| POST | `/api/tools/translate` | Oui | Traduction de texte | Body: `{ text, source_lang?, target_lang, trip_id? }` |
| GET | `/api/tools/currency/rates` | Oui | Taux de change depuis une devise base | `?base=EUR&symbols=USD,THB,JPY` |
| POST | `/api/tools/currency/convert` | Oui | Conversion d'un montant | Body: `{ amount, from, to }` |
| GET | `/api/tools/timezone` | Oui | Info fuseau horaire d'une destination | `?destination=Bangkok&trip_id=xxx` |

---

### 3.7 Module NOTIFICATIONS

| Méthode | Endpoint | Auth | Description | Params clés |
|---|---|---|---|---|
| POST | `/api/notifications/device-token` | Oui | Enregistrement token push (FCM/APNs) | Body: `{ token, platform: 'ios'\|'android' }` |
| DELETE | `/api/notifications/device-token` | Oui | Suppression token (déconnexion) | Body: `{ token }` |
| GET | `/api/notifications/preferences` | Oui | Préférences notifications globales | — |
| PUT | `/api/notifications/preferences` | Oui | Mise à jour préférences globales | Body: `NotificationPreferencesPayload` |
| GET | `/api/notifications/preferences/[tripId]` | Oui | Préférences notifications pour un voyage | — |
| PUT | `/api/notifications/preferences/[tripId]` | Oui | Mise à jour préférences par voyage | — |

---

### 3.8 Module USERS

| Méthode | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/users/preferences` | Oui | Préférences utilisateur (devise, langue, timezone) |
| PUT | `/api/users/preferences` | Oui | Mise à jour préférences |

---

## 4. CODE DES ROUTES CLÉS

### Route 1 — `POST /api/trips` — Création d'un voyage

Cette route est le point d'entrée du cœur métier. Elle gère la validation, la création transactionnelle (voyage + participant admin) et l'initialisation des préférences de notification.

```typescript
// app/api/trips/route.ts
import { NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'
import { createServerClient } from '@/lib/supabase/server'
import { getUserFromRequest } from '@/lib/utils/auth'
import { ApiError, handleApiError } from '@/lib/utils/errors'
import { tripService } from '@/lib/services/tripService'

// Schéma de validation Zod
const TripCreateSchema = z.object({
  name: z.string().min(1).max(100).trim(),
  description: z.string().max(500).trim().optional(),
  destinations: z.array(
    z.object({
      country_code: z.string().length(2).toUpperCase(),
      city: z.string().max(100).trim(),
      primary_currency: z.string().length(3).toUpperCase().optional(),
      primary_language: z.string().max(10).optional(),
      timezone: z.string().max(50).optional(),
    })
  ).min(1).max(10),
  start_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Format YYYY-MM-DD requis'),
  end_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/, 'Format YYYY-MM-DD requis'),
  cover_image_url: z.string().url().optional(),
}).refine(
  (data) => new Date(data.end_date) >= new Date(data.start_date),
  { message: 'La date

— BACKEND ENGINEER

# Livrable Backend Engineer — Travel Companion App
## MVP B2C · Next.js API Routes · Supabase · Anthropic (optionnel)

---

## 1. SCHÉMA DE BASE DE DONNÉES COMPLET

### 1.1 Vue d'ensemble des tables

```
users (géré par Supabase Auth)
profiles
trips
trip_destinations
trip_participants
trip_invitations
itinerary_steps
step_documents
documents
memories
memory_media
albums
album_memories
notification_preferences
device_tokens
exchange_rate_cache
```

### 1.2 Schéma SQL complet

```sql
-- ============================================================
-- EXTENSION
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm"; -- recherche full-text destinations

-- ============================================================
-- TABLE : profiles
-- Complète les données Supabase Auth (auth.users)
-- ============================================================
CREATE TABLE profiles (
  id            UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  display_name  VARCHAR(100),
  avatar_url    TEXT,
  locale        VARCHAR(10)  DEFAULT 'fr',
  currency      CHAR(3)      DEFAULT 'EUR',
  timezone      VARCHAR(50)  DEFAULT 'Europe/Paris',
  created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- ============================================================
-- TABLE : trips
-- ============================================================
CREATE TABLE trips (
  id              UUID         PRIMARY KEY DEFAULT uuid_generate_v4(),
  owner_id        UUID         NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  name            VARCHAR(100) NOT NULL,
  description     TEXT,
  cover_image_url TEXT,
  start_date      DATE         NOT NULL,
  end_date        DATE         NOT NULL,
  status          VARCHAR(20)  NOT NULL DEFAULT 'upcoming'
                               CHECK (status IN ('upcoming','ongoing','past','cancelled')),
  created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  CONSTRAINT valid_dates CHECK (end_date >= start_date)
);

CREATE INDEX idx_trips_owner_id ON trips(owner_id);
CREATE INDEX idx_trips_status   ON trips(status);

-- ============================================================
-- TABLE : trip_destinations
-- Un voyage peut avoir plusieurs destinations
-- ============================================================
CREATE TABLE trip_destinations (
  id               UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id          UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  city             VARCHAR(100) NOT NULL,
  country_code     CHAR(2)     NOT NULL,
  primary_currency CHAR(3),
  primary_language VARCHAR(10),
  timezone         VARCHAR(50),
  sort_order       SMALLINT    NOT NULL DEFAULT 0,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_trip_destinations_trip_id ON trip_destinations(trip_id);

-- ============================================================
-- TABLE : trip_participants
-- Rôles : admin | editor | viewer
-- ============================================================
CREATE TABLE trip_participants (
  id         UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id    UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  user_id    UUID        NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  role       VARCHAR(20) NOT NULL DEFAULT 'viewer'
             CHECK (role IN ('admin','editor','viewer')),
  joined_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (trip_id, user_id)
);

CREATE INDEX idx_trip_participants_trip_id  ON trip_participants(trip_id);
CREATE INDEX idx_trip_participants_user_id  ON trip_participants(user_id);

-- ============================================================
-- TABLE : trip_invitations
-- Lien d'invitation par token ou email
-- ============================================================
CREATE TABLE trip_invitations (
  id          UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id     UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  invited_by  UUID        NOT NULL REFERENCES profiles(id),
  email       VARCHAR(255),                         -- null = invitation par lien
  role        VARCHAR(20) NOT NULL DEFAULT 'viewer'
              CHECK (role IN ('editor','viewer')),
  token       VARCHAR(64) NOT NULL UNIQUE DEFAULT encode(gen_random_bytes(32), 'hex'),
  status      VARCHAR(20) NOT NULL DEFAULT 'pending'
              CHECK (status IN ('pending','accepted','revoked','expired')),
  expires_at  TIMESTAMPTZ NOT NULL DEFAULT (NOW() + INTERVAL '7 days'),
  accepted_at TIMESTAMPTZ,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invitations_token   ON trip_invitations(token);
CREATE INDEX idx_invitations_trip_id ON trip_invitations(trip_id);
CREATE INDEX idx_invitations_email   ON trip_invitations(email);

-- ============================================================
-- TABLE : itinerary_steps
-- ============================================================
CREATE TABLE itinerary_steps (
  id           UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id      UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  step_date    DATE        NOT NULL,
  start_time   TIME,
  end_time     TIME,
  title        VARCHAR(200) NOT NULL,
  description  TEXT,
  location     VARCHAR(300),
  latitude     DECIMAL(9,6),
  longitude    DECIMAL(9,6),
  category     VARCHAR(30) DEFAULT 'other'
               CHECK (category IN ('flight','accommodation','activity','restaurant','transport','other')),
  sort_order   SMALLINT    NOT NULL DEFAULT 0,
  created_by   UUID        NOT NULL REFERENCES profiles(id),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT step_within_trip_dates CHECK (
    step_date >= (SELECT start_date FROM trips WHERE id = trip_id)
    AND step_date <= (SELECT end_date FROM trips WHERE id = trip_id)
  )
);

CREATE INDEX idx_steps_trip_id   ON itinerary_steps(trip_id);
CREATE INDEX idx_steps_step_date ON itinerary_steps(trip_id, step_date);

-- ============================================================
-- TABLE : documents
-- Métadonnées des fichiers stockés dans Supabase Storage
-- ============================================================
CREATE TABLE documents (
  id            UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id       UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  uploaded_by   UUID        NOT NULL REFERENCES profiles(id),
  filename      VARCHAR(255) NOT NULL,
  storage_path  TEXT        NOT NULL UNIQUE,  -- path dans Supabase Storage
  mime_type     VARCHAR(100) NOT NULL,
  size_bytes    INTEGER     NOT NULL CHECK (size_bytes <= 20971520), -- 20 Mo max
  category      VARCHAR(30) NOT NULL DEFAULT 'other'
                CHECK (category IN ('flight','accommodation','activity','other')),
  upload_status VARCHAR(20) NOT NULL DEFAULT 'pending'
                CHECK (upload_status IN ('pending','confirmed','failed')),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_documents_trip_id  ON documents(trip_id);
CREATE INDEX idx_documents_category ON documents(trip_id, category);

-- ============================================================
-- TABLE : step_documents
-- Association N:N étape ↔ document
-- ============================================================
CREATE TABLE step_documents (
  step_id     UUID NOT NULL REFERENCES itinerary_steps(id) ON DELETE CASCADE,
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  PRIMARY KEY (step_id, document_id)
);

-- ============================================================
-- TABLE : memories
-- Souvenirs texte + lieu, les médias sont dans memory_media
-- ============================================================
CREATE TABLE memories (
  id          UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id     UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  author_id   UUID        NOT NULL REFERENCES profiles(id),
  title       VARCHAR(200),
  note        TEXT,
  location    VARCHAR(300),
  latitude    DECIMAL(9,6),
  longitude   DECIMAL(9,6),
  memory_date DATE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_memories_trip_id   ON memories(trip_id);
CREATE INDEX idx_memories_author_id ON memories(author_id);

-- ============================================================
-- TABLE : memory_media
-- Fichiers (photos, vidéos) associés à un souvenir
-- ============================================================
CREATE TABLE memory_media (
  id           UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  memory_id    UUID        NOT NULL REFERENCES memories(id) ON DELETE CASCADE,
  storage_path TEXT        NOT NULL UNIQUE,
  mime_type    VARCHAR(100) NOT NULL,
  size_bytes   INTEGER     NOT NULL CHECK (size_bytes <= 20971520),
  width        INTEGER,
  height       INTEGER,
  sort_order   SMALLINT    NOT NULL DEFAULT 0,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_memory_media_memory_id ON memory_media(memory_id);

-- ============================================================
-- TABLE : albums
-- Albums partageables publiquement
-- ============================================================
CREATE TABLE albums (
  id           UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id      UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  created_by   UUID        NOT NULL REFERENCES profiles(id),
  title        VARCHAR(200) NOT NULL,
  description  TEXT,
  slug         VARCHAR(80) NOT NULL UNIQUE DEFAULT encode(gen_random_bytes(8), 'hex'),
  is_public    BOOLEAN     NOT NULL DEFAULT false,
  expires_at   TIMESTAMPTZ,                           -- null = pas d'expiration
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_albums_trip_id ON albums(trip_id);
CREATE UNIQUE INDEX idx_albums_slug ON albums(slug);

-- ============================================================
-- TABLE : album_memories
-- Association N:N album ↔ souvenir
-- ============================================================
CREATE TABLE album_memories (
  album_id    UUID     NOT NULL REFERENCES albums(id) ON DELETE CASCADE,
  memory_id   UUID     NOT NULL REFERENCES memories(id) ON DELETE CASCADE,
  sort_order  SMALLINT NOT NULL DEFAULT 0,
  added_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (album_id, memory_id)
);

-- ============================================================
-- TABLE : device_tokens
-- Tokens push par utilisateur/plateforme
-- ============================================================
CREATE TABLE device_tokens (
  id          UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id     UUID        NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  token       TEXT        NOT NULL,
  platform    VARCHAR(10) NOT NULL CHECK (platform IN ('ios','android','web')),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_seen   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (user_id, token)
);

CREATE INDEX idx_device_tokens_user_id ON device_tokens(user_id);

-- ============================================================
-- TABLE : notification_preferences
-- Préférences par utilisateur, avec override optionnel par voyage
-- ============================================================
CREATE TABLE notification_preferences (
  id                    UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id               UUID        NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  trip_id               UUID        REFERENCES trips(id) ON DELETE CASCADE, -- null = global
  step_reminder_enabled BOOLEAN     NOT NULL DEFAULT true,
  reminder_delay_hours  SMALLINT[]  NOT NULL DEFAULT '{24,2}', -- délais en heures
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (user_id, trip_id)   -- (user_id, null) = préférences globales
);

-- ============================================================
-- TABLE : exchange_rate_cache
-- Cache des taux de change (TTL géré applicativement)
-- ============================================================
CREATE TABLE exchange_rate_cache (
  base_currency CHAR(3)     NOT NULL,
  rates         JSONB       NOT NULL,  -- { "USD": 1.08, "THB": 38.5, ... }
  fetched_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (base_currency)
);
```

---

### 1.3 Row Level Security (RLS)

```sql
-- ============================================================
-- RLS — ACTIVATION
-- ============================================================
ALTER TABLE profiles               ENABLE ROW LEVEL SECURITY;
ALTER TABLE trips                  ENABLE ROW LEVEL SECURITY;
ALTER TABLE trip_destinations      ENABLE ROW LEVEL SECURITY;
ALTER TABLE trip_participants      ENABLE ROW LEVEL SECURITY;
ALTER TABLE trip_invitations       ENABLE ROW LEVEL SECURITY;
ALTER TABLE itinerary_steps        ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents              ENABLE ROW LEVEL SECURITY;
ALTER TABLE step_documents         ENABLE ROW LEVEL SECURITY;
ALTER TABLE memories               ENABLE ROW LEVEL SECURITY;
ALTER TABLE memory_media           ENABLE ROW LEVEL SECURITY;
ALTER TABLE albums                 ENABLE ROW LEVEL SECURITY;
ALTER TABLE album_memories         ENABLE ROW LEVEL SECURITY;
ALTER TABLE device_tokens          ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_preferences ENABLE ROW LEVEL SECURITY;

-- ============================================================
-- FONCTION HELPER — rôle d'un utilisateur sur un voyage
-- ============================================================
CREATE OR REPLACE FUNCTION get_trip_role(p_trip_id UUID, p_user_id UUID)
RETURNS VARCHAR AS $$
  SELECT role FROM trip_participants
  WHERE trip_id = p_trip_id AND user_id = p_user_id
$$ LANGUAGE SQL SECURITY DEFINER STABLE;

-- ============================================================
-- RLS — PROFILES
-- ============================================================
CREATE POLICY "profiles_select_own"
  ON profiles FOR SELECT
  USING (id = auth.uid());

CREATE POLICY "profiles_update_own"
  ON profiles FOR UPDATE
  USING (id = auth.uid());

-- ============================================================
-- RLS — TRIPS
-- ============================================================
CREATE POLICY "trips_select_participant"
  ON trips FOR SELECT
  USING (
    id IN (
      SELECT trip_id FROM trip_participants WHERE user_id = auth.uid()
    )
  );

CREATE POLICY "trips_insert_authenticated"
  ON trips FOR INSERT
  WITH CHECK (owner_id = auth.uid());

CREATE POLICY "trips_update_admin"
  ON trips FOR UPDATE
  USING (get_trip_role(id, auth.uid()) = 'admin');

CREATE POLICY "trips_delete_owner"
  ON trips FOR DELETE
  USING (owner_id = auth.uid());

-- ============================================================
-- RLS — TRIP_PARTICIPANTS
-- ============================================================
CREATE POLICY "participants_select_participant"
  ON trip_participants FOR SELECT
  USING (get_trip_role(trip_id, auth.uid()) IS NOT NULL);

CREATE POLICY "participants_update_admin"
  ON trip_participants FOR UPDATE
  USING (get_trip_role(trip_id, auth.uid()) = 'admin');

CREATE POLICY "participants_delete_admin_or_self"
  ON trip_participants FOR DELETE
  USING (
    get_trip_role(trip_id, auth.uid()) = 'admin'
    OR user_id = auth.uid()
  );

-- ============================================================
-- RLS — ITINERARY_STEPS
-- ============================================================
CREATE POLICY "steps_select_participant"
  ON itinerary_steps FOR SELECT
  USING (get_trip_role(trip_id, auth.uid()) IS NOT NULL);

CREATE POLICY "steps_insert_editor"
  ON itinerary_steps FOR INSERT
  WITH CHECK (
    get_trip_role(trip_id, auth.uid()) IN ('admin','editor')
    AND created_by = auth.uid()
  );

CREATE POLICY "steps_update_editor"

— BACKEND ENGINEER

# Livrable Backend Engineer — Travel Companion App
## MVP B2C · Next.js API Routes · Supabase · Anthropic

---

> **Note de contexte** : Ce livrable s'appuie sur les décisions architecturales des livrables précédents (Architecte, Tech Lead, Frontend). Il adapte la stack à **Next.js API Routes + Supabase** conformément au brief Backend Engineer, tout en restant cohérent avec le schéma de données et les modules déjà définis. Le schéma SQL complet ayant été produit dans la session précédente, ce livrable reprend les éléments manquants, finalise la stratégie RLS, et se concentre sur l'architecture des services, les endpoints et le code des routes clés.

---

## 1. SCHÉMA DE BASE DE DONNÉES — COMPLÉMENTS ET FINALISATION

### 1.1 Tables manquantes (complément au schéma existant)

```sql
-- ============================================================
-- EXTENSION : activation uuid
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================================
-- TABLE : profiles
-- Extension de auth.users Supabase
-- ============================================================
CREATE TABLE profiles (
  id            UUID        PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email         VARCHAR(255) NOT NULL UNIQUE,
  full_name     VARCHAR(200),
  avatar_url    TEXT,
  default_currency CHAR(3)  NOT NULL DEFAULT 'EUR',
  default_locale   VARCHAR(10) NOT NULL DEFAULT 'fr',
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- TABLE : trips
-- Voyage principal
-- ============================================================
CREATE TABLE trips (
  id            UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  owner_id      UUID        NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  name          VARCHAR(200) NOT NULL,
  cover_url     TEXT,
  status        VARCHAR(20) NOT NULL DEFAULT 'upcoming'
                CHECK (status IN ('upcoming','ongoing','past','archived')),
  start_date    DATE        NOT NULL,
  end_date      DATE        NOT NULL,
  CHECK (end_date >= start_date),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_trips_owner_id ON trips(owner_id);
CREATE INDEX idx_trips_status   ON trips(status);

-- ============================================================
-- TABLE : trip_destinations
-- Destinations multiples par voyage (N:1 trip)
-- ============================================================
CREATE TABLE trip_destinations (
  id            UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id       UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  name          VARCHAR(200) NOT NULL,
  country_code  CHAR(2),
  currency_code CHAR(3),
  timezone      VARCHAR(80),
  latitude      DECIMAL(9,6),
  longitude     DECIMAL(9,6),
  sort_order    SMALLINT    NOT NULL DEFAULT 0,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_trip_destinations_trip_id ON trip_destinations(trip_id);

-- ============================================================
-- TABLE : trip_participants
-- Membres d'un voyage avec rôle
-- ============================================================
CREATE TABLE trip_participants (
  trip_id     UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  user_id     UUID        NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  role        VARCHAR(20) NOT NULL DEFAULT 'viewer'
              CHECK (role IN ('admin','editor','viewer')),
  joined_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (trip_id, user_id)
);

CREATE INDEX idx_trip_participants_user_id ON trip_participants(user_id);

-- ============================================================
-- TABLE : trip_invitations
-- Invitations en attente (email ou lien)
-- ============================================================
CREATE TABLE trip_invitations (
  id          UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id     UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  invited_by  UUID        NOT NULL REFERENCES profiles(id),
  email       VARCHAR(255),
  token       VARCHAR(80) NOT NULL UNIQUE DEFAULT encode(gen_random_bytes(16), 'hex'),
  role        VARCHAR(20) NOT NULL DEFAULT 'viewer'
              CHECK (role IN ('editor','viewer')),
  status      VARCHAR(20) NOT NULL DEFAULT 'pending'
              CHECK (status IN ('pending','accepted','expired','revoked')),
  expires_at  TIMESTAMPTZ NOT NULL DEFAULT NOW() + INTERVAL '7 days',
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_invitations_trip_id ON trip_invitations(trip_id);
CREATE INDEX idx_invitations_token   ON trip_invitations(token);
CREATE INDEX idx_invitations_email   ON trip_invitations(email);

-- ============================================================
-- TABLE : itinerary_steps
-- Étapes de l'itinéraire
-- ============================================================
CREATE TABLE itinerary_steps (
  id            UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id       UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  created_by    UUID        NOT NULL REFERENCES profiles(id),
  title         VARCHAR(200) NOT NULL,
  step_type     VARCHAR(30) NOT NULL DEFAULT 'other'
                CHECK (step_type IN ('flight','accommodation','activity','restaurant','transport','other')),
  step_date     DATE        NOT NULL,
  start_time    TIME,
  end_time      TIME,
  location_name VARCHAR(300),
  address       TEXT,
  latitude      DECIMAL(9,6),
  longitude     DECIMAL(9,6),
  notes         TEXT,
  timezone      VARCHAR(80),                          -- fuseau de l'étape pour notifications
  sort_order    SMALLINT    NOT NULL DEFAULT 0,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_steps_trip_id   ON itinerary_steps(trip_id);
CREATE INDEX idx_steps_step_date ON itinerary_steps(step_date);

-- ============================================================
-- TABLE : documents
-- Fichiers importés (PDF, images)
-- ============================================================
CREATE TABLE documents (
  id            UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id       UUID        NOT NULL REFERENCES trips(id) ON DELETE CASCADE,
  uploaded_by   UUID        NOT NULL REFERENCES profiles(id),
  filename      VARCHAR(255) NOT NULL,
  storage_path  TEXT        NOT NULL UNIQUE,
  mime_type     VARCHAR(100) NOT NULL,
  size_bytes    INTEGER     NOT NULL CHECK (size_bytes <= 20971520), -- 20 Mo max
  category      VARCHAR(30) NOT NULL DEFAULT 'other'
                CHECK (category IN ('flight','accommodation','activity','transport','other')),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_documents_trip_id ON documents(trip_id);
CREATE INDEX idx_documents_category ON documents(category);

-- ============================================================
-- TABLE : step_documents
-- Association N:N étape ↔ document
-- ============================================================
CREATE TABLE step_documents (
  step_id     UUID NOT NULL REFERENCES itinerary_steps(id) ON DELETE CASCADE,
  document_id UUID NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
  PRIMARY KEY (step_id, document_id)
);

-- ============================================================
-- TABLE : scheduled_notifications
-- Notifications planifiées pour les rappels d'étapes
-- ============================================================
CREATE TABLE scheduled_notifications (
  id              UUID        PRIMARY KEY DEFAULT uuid_generate_v4(),
  step_id         UUID        NOT NULL REFERENCES itinerary_steps(id) ON DELETE CASCADE,
  user_id         UUID        NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  scheduled_at    TIMESTAMPTZ NOT NULL,               -- moment d'envoi calculé
  sent_at         TIMESTAMPTZ,                        -- null = pas encore envoyé
  status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','sent','failed','cancelled')),
  payload         JSONB       NOT NULL DEFAULT '{}'
);

CREATE INDEX idx_scheduled_notif_step_id     ON scheduled_notifications(step_id);
CREATE INDEX idx_scheduled_notif_user_id     ON scheduled_notifications(user_id);
CREATE INDEX idx_scheduled_notif_scheduled   ON scheduled_notifications(scheduled_at)
  WHERE status = 'pending';
```

### 1.2 RLS — Finalisation des politiques

```sql
-- ============================================================
-- RLS — ITINERARY_STEPS (suite)
-- ============================================================
CREATE POLICY "steps_update_editor"
  ON itinerary_steps FOR UPDATE
  USING (get_trip_role(trip_id, auth.uid()) IN ('admin','editor'));

CREATE POLICY "steps_delete_admin"
  ON itinerary_steps FOR DELETE
  USING (get_trip_role(trip_id, auth.uid()) = 'admin');

-- ============================================================
-- RLS — DOCUMENTS
-- ============================================================
CREATE POLICY "documents_select_participant"
  ON documents FOR SELECT
  USING (get_trip_role(trip_id, auth.uid()) IS NOT NULL);

CREATE POLICY "documents_insert_editor"
  ON documents FOR INSERT
  WITH CHECK (
    get_trip_role(trip_id, auth.uid()) IN ('admin','editor')
    AND uploaded_by = auth.uid()
  );

CREATE POLICY "documents_delete_admin_or_uploader"
  ON documents FOR DELETE
  USING (
    get_trip_role(trip_id, auth.uid()) = 'admin'
    OR uploaded_by = auth.uid()
  );

-- ============================================================
-- RLS — MEMORIES
-- ============================================================
CREATE POLICY "memories_select_participant"
  ON memories FOR SELECT
  USING (get_trip_role(trip_id, auth.uid()) IS NOT NULL);

CREATE POLICY "memories_insert_participant"
  ON memories FOR INSERT
  WITH CHECK (
    get_trip_role(trip_id, auth.uid()) IS NOT NULL
    AND author_id = auth.uid()
  );

CREATE POLICY "memories_update_author"
  ON memories FOR UPDATE
  USING (author_id = auth.uid());

CREATE POLICY "memories_delete_admin_or_author"
  ON memories FOR DELETE
  USING (
    get_trip_role(trip_id, auth.uid()) = 'admin'
    OR author_id = auth.uid()
  );

-- ============================================================
-- RLS — MEMORY_MEDIA
-- ============================================================
CREATE POLICY "memory_media_select_via_memory"
  ON memory_media FOR SELECT
  USING (
    memory_id IN (
      SELECT m.id FROM memories m
      WHERE get_trip_role(m.trip_id, auth.uid()) IS NOT NULL
    )
  );

-- ============================================================
-- RLS — ALBUMS (accès public via slug géré applicativement)
-- ============================================================
CREATE POLICY "albums_select_participant"
  ON albums FOR SELECT
  USING (get_trip_role(trip_id, auth.uid()) IS NOT NULL);

CREATE POLICY "albums_insert_participant"
  ON albums FOR INSERT
  WITH CHECK (
    get_trip_role(trip_id, auth.uid()) IN ('admin','editor')
    AND created_by = auth.uid()
  );

CREATE POLICY "albums_update_creator_or_admin"
  ON albums FOR UPDATE
  USING (
    created_by = auth.uid()
    OR get_trip_role(trip_id, auth.uid()) = 'admin'
  );

CREATE POLICY "albums_delete_creator_or_admin"
  ON albums FOR DELETE
  USING (
    created_by = auth.uid()
    OR get_trip_role(trip_id, auth.uid()) = 'admin'
  );

-- ============================================================
-- RLS — DEVICE_TOKENS
-- ============================================================
CREATE POLICY "device_tokens_own"
  ON device_tokens FOR ALL
  USING (user_id = auth.uid())
  WITH CHECK (user_id = auth.uid());

-- ============================================================
-- RLS — NOTIFICATION_PREFERENCES
-- ============================================================
CREATE POLICY "notif_prefs_own"
  ON notification_preferences FOR ALL
  USING (user_id = auth.uid())
  WITH CHECK (user_id = auth.uid());

-- ============================================================
-- TRIGGER : mise à jour automatique de updated_at
-- ============================================================
CREATE OR REPLACE FUNCTION trigger_set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER set_updated_at BEFORE UPDATE ON trips
  FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON profiles
  FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON itinerary_steps
  FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON albums
  FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
```

---

## 2. ARCHITECTURE DES SERVICES BACKEND

### 2.1 Structure des dossiers — Next.js API Routes

```
src/
├── app/
│   └── api/                              # API Routes Next.js 14
│       ├── auth/
│       │   ├── register/route.ts
│       │   ├── login/route.ts
│       │   └── me/route.ts
│       ├── trips/
│       │   ├── route.ts                  # GET (liste) / POST (création)
│       │   └── [tripId]/
│       │       ├── route.ts              # GET / PATCH / DELETE
│       │       ├── participants/
│       │       │   ├── route.ts          # GET liste / POST invitation
│       │       │   └── [userId]/route.ts # PATCH rôle / DELETE
│       │       ├── invitations/
│       │       │   ├── route.ts          # POST création invitation
│       │       │   └── [token]/
│       │       │       └── accept/route.ts
│       │       ├── steps/
│       │       │   ├── route.ts          # GET liste / POST création
│       │       │   └── [stepId]/
│       │       │       ├── route.ts      # GET / PATCH / DELETE
│       │       │       └── documents/route.ts
│       │       ├── documents/
│       │       │   ├── route.ts          # GET liste catégorisée
│       │       │   └── upload/route.ts   # POST upload (multipart)
│       │       ├── memories/
│       │       │   ├── route.ts          # GET / POST
│       │       │   └── [memoryId]/route.ts
│       │       └── albums/
│       │           ├── route.ts          # GET / POST
│       │           └── [albumId]/
│       │               ├── route.ts      # GET / PATCH / DELETE
│       │               └── memories/route.ts
│       ├── documents/
│       │   └── [documentId]/
│       │       ├── route.ts              # GET (URL signée) / DELETE
│       │       └── download/route.ts
│       ├── albums/
│       │   └── [slug]/
│       │       └── route.ts              # GET public (sans auth)
│       ├── tools/
│       │   ├── translate/route.ts        # POST traduction
│       │   ├── exchange-rates/route.ts   # GET taux de change
│       │   └── timezones/route.ts        # GET fuseaux horaires
│       ├── notifications/
│       │   ├── devices/route.ts          # POST enregistrement token
│       │   └

— BACKEND ENGINEER

*(continuation exacte)*

```
│       └── preferences/route.ts      # GET / PATCH préférences
└── lib/
    ├── supabase/
    │   ├── client.ts                 # Client Supabase server-side
    │   └── admin.ts                  # Client admin (service role)
    ├── middleware/
    │   ├── auth.ts                   # Validation JWT Supabase
    │   ├── rateLimit.ts              # Rate limiting par IP/user
    │   └── validate.ts               # Validation Zod des payloads
    ├── services/
    │   ├── tripService.ts
    │   ├── documentService.ts
    │   ├── memoryService.ts
    │   ├── notificationService.ts
    │   └── toolsService.ts
    └── cache/
        └── redis.ts                  # Client Redis + helpers
```

---

## 3. LISTE COMPLÈTE DES ENDPOINTS API

### Convention générale
- Toutes les routes sont préfixées `/api/`
- Authentification : header `Authorization: Bearer <jwt>` sauf routes publiques
- Réponses : `{ data, error, meta }` pour les listes, `{ data, error }` pour le reste
- Erreurs : `{ error: { code, message, details? } }`
- Pagination : `?page=1&limit=20` sur toutes les listes

---

### 3.1 Auth

| Méthode | Route | Auth | Description | Réponse |
|---|---|---|---|---|
| POST | `/api/auth/register` | Non | Inscription email/password | `{ data: { user, session } }` |
| POST | `/api/auth/login` | Non | Connexion email/password | `{ data: { user, session } }` |
| GET | `/api/auth/me` | Oui | Profil utilisateur courant | `{ data: { user } }` |
| PATCH | `/api/auth/me` | Oui | Mise à jour profil | `{ data: { user } }` |

> Note : OAuth Google/Apple géré nativement par Supabase Auth — pas de route custom nécessaire.

---

### 3.2 Voyages (Trips)

| Méthode | Route | Auth | Description | Réponse |
|---|---|---|---|---|
| GET | `/api/trips` | Oui | Liste des voyages de l'utilisateur | `{ data: Trip[], meta: { total } }` |
| POST | `/api/trips` | Oui | Créer un voyage | `{ data: Trip }` |
| GET | `/api/trips/:tripId` | Oui | Détail d'un voyage | `{ data: Trip }` |
| PATCH | `/api/trips/:tripId` | Oui (admin) | Modifier un voyage | `{ data: Trip }` |
| DELETE | `/api/trips/:tripId` | Oui (admin) | Supprimer un voyage | `{ data: null }` |

---

### 3.3 Participants & Invitations

| Méthode | Route | Auth | Description | Réponse |
|---|---|---|---|---|
| GET | `/api/trips/:tripId/participants` | Oui | Liste des participants | `{ data: Participant[] }` |
| POST | `/api/trips/:tripId/invitations` | Oui (admin/editor) | Créer une invitation | `{ data: { token, expires_at } }` |
| GET | `/api/trips/:tripId/invitations` | Oui (admin) | Liste des invitations actives | `{ data: Invitation[] }` |
| POST | `/api/trips/:tripId/invitations/:token/accept` | Oui | Accepter une invitation | `{ data: Participant }` |
| PATCH | `/api/trips/:tripId/participants/:userId` | Oui (admin) | Changer le rôle | `{ data: Participant }` |
| DELETE | `/api/trips/:tripId/participants/:userId` | Oui (admin) | Retirer un participant | `{ data: null }` |

---

### 3.4 Itinéraire & Étapes

| Méthode | Route | Auth | Description | Réponse |
|---|---|---|---|---|
| GET | `/api/trips/:tripId/steps` | Oui | Liste des étapes (optionnel: `?date=YYYY-MM-DD`) | `{ data: Step[] }` |
| POST | `/api/trips/:tripId/steps` | Oui (admin/editor) | Créer une étape | `{ data: Step }` |
| GET | `/api/trips/:tripId/steps/:stepId` | Oui | Détail d'une étape | `{ data: Step }` |
| PATCH | `/api/trips/:tripId/steps/:stepId` | Oui (admin/editor) | Modifier une étape | `{ data: Step }` |
| DELETE | `/api/trips/:tripId/steps/:stepId` | Oui (admin) | Supprimer une étape | `{ data: null }` |
| GET | `/api/trips/:tripId/steps/:stepId/documents` | Oui | Documents liés à une étape | `{ data: Document[] }` |
| POST | `/api/trips/:tripId/steps/:stepId/documents` | Oui (admin/editor) | Associer un doc à une étape | `{ data: StepDocument }` |
| DELETE | `/api/trips/:tripId/steps/:stepId/documents/:docId` | Oui (admin/editor) | Désassocier un doc | `{ data: null }` |

---

### 3.5 Documents

| Méthode | Route | Auth | Description | Réponse |
|---|---|---|---|---|
| GET | `/api/trips/:tripId/documents` | Oui | Liste documents (optionnel: `?category=flight`) | `{ data: Document[] }` |
| POST | `/api/trips/:tripId/documents/upload` | Oui (admin/editor) | Upload fichier (multipart/form-data) | `{ data: Document }` |
| GET | `/api/documents/:documentId` | Oui | Métadonnées document | `{ data: Document }` |
| GET | `/api/documents/:documentId/download` | Oui | URL signée de téléchargement | `{ data: { url, expires_at } }` |
| PATCH | `/api/documents/:documentId` | Oui (admin/uploader) | Modifier catégorie ou nom | `{ data: Document }` |
| DELETE | `/api/documents/:documentId` | Oui (admin/uploader) | Supprimer document | `{ data: null }` |

---

### 3.6 Souvenirs & Albums

| Méthode | Route | Auth | Description | Réponse |
|---|---|---|---|---|
| GET | `/api/trips/:tripId/memories` | Oui | Liste des souvenirs | `{ data: Memory[], meta }` |
| POST | `/api/trips/:tripId/memories` | Oui | Créer un souvenir | `{ data: Memory }` |
| GET | `/api/trips/:tripId/memories/:memoryId` | Oui | Détail souvenir | `{ data: Memory }` |
| PATCH | `/api/trips/:tripId/memories/:memoryId` | Oui (auteur) | Modifier un souvenir | `{ data: Memory }` |
| DELETE | `/api/trips/:tripId/memories/:memoryId` | Oui (auteur/admin) | Supprimer un souvenir | `{ data: null }` |
| GET | `/api/trips/:tripId/albums` | Oui | Liste albums du voyage | `{ data: Album[] }` |
| POST | `/api/trips/:tripId/albums` | Oui (admin/editor) | Créer un album | `{ data: Album }` |
| GET | `/api/trips/:tripId/albums/:albumId` | Oui | Détail album | `{ data: Album }` |
| PATCH | `/api/trips/:tripId/albums/:albumId` | Oui (créateur/admin) | Modifier album | `{ data: Album }` |
| DELETE | `/api/trips/:tripId/albums/:albumId` | Oui (créateur/admin) | Supprimer album | `{ data: null }` |
| POST | `/api/trips/:tripId/albums/:albumId/memories` | Oui | Ajouter souvenirs à l'album | `{ data: Album }` |
| DELETE | `/api/trips/:tripId/albums/:albumId/memories/:memoryId` | Oui | Retirer souvenir de l'album | `{ data: null }` |
| GET | `/api/albums/:slug` | **Non** | Vue publique album (lien partagé) | `{ data: PublicAlbum }` |

---

### 3.7 Outils

| Méthode | Route | Auth | Description | Réponse |
|---|---|---|---|---|
| POST | `/api/tools/translate` | Oui | Traduire du texte | `{ data: { translated_text, detected_source } }` |
| GET | `/api/tools/exchange-rates` | Oui | Taux de change (`?base=EUR&targets=USD,JPY`) | `{ data: { base, rates, updated_at } }` |
| GET | `/api/tools/timezones` | Oui | Heure destination (`?timezone=Asia/Tokyo`) | `{ data: { timezone, current_time, offset } }` |

---

### 3.8 Notifications

| Méthode | Route | Auth | Description | Réponse |
|---|---|---|---|---|
| POST | `/api/notifications/devices` | Oui | Enregistrer device token | `{ data: DeviceToken }` |
| DELETE | `/api/notifications/devices/:deviceId` | Oui | Supprimer device token | `{ data: null }` |
| GET | `/api/notifications/preferences` | Oui | Récupérer préférences | `{ data: NotificationPreferences }` |
| PATCH | `/api/notifications/preferences` | Oui | Mettre à jour préférences | `{ data: NotificationPreferences }` |

---

## 4. CODE DES ROUTES CLÉS

### 4.1 Upload de document (`POST /api/trips/:tripId/documents/upload`)

Route la plus complexe : validation multipart, vérification quota, upload S3, création enregistrement, planification du cache offline.

```typescript
// src/app/api/trips/[tripId]/documents/upload/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/client';
import { createAdminClient } from '@/lib/supabase/admin';
import { requireAuth } from '@/lib/middleware/auth';
import { requireTripRole } from '@/lib/middleware/tripRole';
import { rateLimit } from '@/lib/middleware/rateLimit';
import { nanoid } from 'nanoid';

const MAX_FILE_SIZE = 20 * 1024 * 1024; // 20 Mo
const ALLOWED_MIME_TYPES = ['application/pdf', 'image/jpeg', 'image/png'];
const ALLOWED_CATEGORIES = ['flight', 'accommodation', 'activity', 'other'] as const;

type DocumentCategory = typeof ALLOWED_CATEGORIES[number];

export async function POST(
  request: NextRequest,
  { params }: { params: { tripId: string } }
) {
  // 1. Rate limiting — 20 uploads / 15 min par utilisateur
  const rateLimitResult = await rateLimit(request, {
    key: 'document-upload',
    limit: 20,
    window: 15 * 60,
  });
  if (!rateLimitResult.success) {
    return NextResponse.json(
      { error: { code: 'RATE_LIMIT_EXCEEDED', message: 'Too many uploads. Try again later.' } },
      { status: 429 }
    );
  }

  // 2. Authentification
  const supabase = createClient(request);
  const { user, error: authError } = await requireAuth(supabase);
  if (authError || !user) {
    return NextResponse.json(
      { error: { code: 'UNAUTHORIZED', message: 'Authentication required' } },
      { status: 401 }
    );
  }

  // 3. Vérification rôle sur le voyage (admin ou editor)
  const { role, error: roleError } = await requireTripRole(supabase, params.tripId, user.id, ['admin', 'editor']);
  if (roleError) {
    return NextResponse.json(
      { error: { code: 'FORBIDDEN', message: 'Insufficient permissions' } },
      { status: 403 }
    );
  }

  // 4. Parsing multipart
  let formData: FormData;
  try {
    formData = await request.formData();
  } catch {
    return NextResponse.json(
      { error: { code: 'INVALID_BODY', message: 'Expected multipart/form-data' } },
      { status: 400 }
    );
  }

  const file = formData.get('file') as File | null;
  const category = (formData.get('category') as string) || 'other';
  const stepId = formData.get('step_id') as string | null;
  const customName = formData.get('name') as string | null;

  // 5. Validation fichier
  if (!file) {
    return NextResponse.json(
      { error: { code: 'MISSING_FILE', message: 'No file provided' } },
      { status: 400 }
    );
  }

  if (file.size > MAX_FILE_SIZE) {
    return NextResponse.json(
      { error: { code: 'FILE_TOO_LARGE', message: 'File exceeds 20MB limit' } },
      { status: 413 }
    );
  }

  if (!ALLOWED_MIME_TYPES.includes(file.type)) {
    return NextResponse.json(
      { error: { code: 'INVALID_FILE_TYPE', message: 'Allowed types: PDF, JPG, PNG' } },
      { status: 415 }
    );
  }

  if (!ALLOWED_CATEGORIES.includes(category as DocumentCategory)) {
    return NextResponse.json(
      { error: { code: 'INVALID_CATEGORY', message: `Category must be one of: ${ALLOWED_CATEGORIES.join(', ')}` } },
      { status: 400 }
    );
  }

  // 6. Vérification quota utilisateur (max 500 Mo par voyage)
  const { data: quotaData } = await supabase
    .from('documents')
    .select('file_size')
    .eq('trip_id', params.tripId);

  const currentUsage = quotaData?.reduce((sum, doc) => sum + doc.file_size, 0) ?? 0;
  const TRIP_QUOTA = 500 * 1024 * 1024; // 500 Mo par voyage

  if (currentUsage + file.size > TRIP_QUOTA) {
    return NextResponse.json(
      { error: { code: 'QUOTA_EXCEEDED', message: 'Storage quota exceeded for this trip (500MB)' } },
      { status: 507 }
    );
  }

  // 7. Si step_id fourni, vérifier qu'il appartient bien au trip
  if (stepId) {
    const { data: step } = await supabase
      .from('itinerary_steps')
      .select('id')
      .eq('id', stepId)
      .eq('trip_id', params.tripId)
      .single();

    if (!step) {
      return NextResponse.json(
        { error: { code: 'STEP_NOT_FOUND',

— BACKEND ENGINEER

*(continuation exacte à partir de l'interruption)*

```typescript
        message: 'Step not found or does not belong to this trip' } },
        { status: 404 }
      );
    }
  }

  // 8. Upload vers Supabase Storage
  const adminClient = createAdminClient();
  const fileExtension = file.name.split('.').pop() ?? 'bin';
  const storageKey = `trips/${params.tripId}/documents/${nanoid()}.${fileExtension}`;
  const fileBuffer = await file.arrayBuffer();

  const { error: uploadError } = await adminClient.storage
    .from('travel-documents')
    .upload(storageKey, fileBuffer, {
      contentType: file.type,
      upsert: false,
    });

  if (uploadError) {
    console.error('[document-upload] Storage error:', uploadError);
    return NextResponse.json(
      { error: { code: 'STORAGE_ERROR', message: 'Failed to upload file' } },
      { status: 500 }
    );
  }

  // 9. Enregistrement en base de données
  const documentName = customName?.trim() || file.name;
  const { data: document, error: dbError } = await supabase
    .from('documents')
    .insert({
      trip_id: params.tripId,
      step_id: stepId ?? null,
      uploaded_by: user.id,
      name: documentName,
      category: category as DocumentCategory,
      file_size: file.size,
      mime_type: file.type,
      storage_key: storageKey,
    })
    .select()
    .single();

  if (dbError) {
    // Rollback : supprimer le fichier uploadé si l'insertion BDD échoue
    await adminClient.storage
      .from('travel-documents')
      .remove([storageKey]);

    console.error('[document-upload] DB insert error:', dbError);
    return NextResponse.json(
      { error: { code: 'DATABASE_ERROR', message: 'Failed to save document metadata' } },
      { status: 500 }
    );
  }

  return NextResponse.json({ data: document }, { status: 201 });
}
```

---

### 4.2 Vue publique d'un album (`GET /api/albums/:slug`)

Route non authentifiée, exposée publiquement. Sécurité maximale : pas de fuite de données privées, contrôle expiration lien.

```typescript
// src/app/api/albums/[slug]/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { createAdminClient } from '@/lib/supabase/admin';
import { rateLimit } from '@/lib/middleware/rateLimit';

export async function GET(
  request: NextRequest,
  { params }: { params: { slug: string } }
) {
  // 1. Rate limiting renforcé sur route publique — 60 req / min par IP
  const rateLimitResult = await rateLimit(request, {
    key: 'public-album',
    limit: 60,
    window: 60,
    identifier: 'ip',
  });
  if (!rateLimitResult.success) {
    return NextResponse.json(
      { error: { code: 'RATE_LIMIT_EXCEEDED', message: 'Too many requests' } },
      { status: 429 }
    );
  }

  const adminClient = createAdminClient();

  // 2. Récupération de l'album via son slug public
  const { data: album, error: albumError } = await adminClient
    .from('memory_albums')
    .select(`
      id,
      title,
      description,
      slug,
      is_public,
      share_expires_at,
      created_at,
      trip:trips (
        name,
        destination_label
      )
    `)
    .eq('slug', params.slug)
    .single();

  // Réponse identique que l'album n'existe pas ou soit privé
  // (évite l'énumération des slugs)
  if (albumError || !album) {
    return NextResponse.json(
      { error: { code: 'NOT_FOUND', message: 'Album not found' } },
      { status: 404 }
    );
  }

  if (!album.is_public) {
    return NextResponse.json(
      { error: { code: 'NOT_FOUND', message: 'Album not found' } },
      { status: 404 }
    );
  }

  // 3. Vérification expiration du lien
  if (album.share_expires_at && new Date(album.share_expires_at) < new Date()) {
    return NextResponse.json(
      { error: { code: 'LINK_EXPIRED', message: 'This sharing link has expired' } },
      { status: 410 }
    );
  }

  // 4. Récupération des souvenirs de l'album
  // On ne retourne que les champs safe (jamais user_id, données internes)
  const { data: memories, error: memoriesError } = await adminClient
    .from('album_memories')
    .select(`
      memory:memories (
        id,
        title,
        note,
        taken_at,
        location_label,
        media_url,
        media_type
      )
    `)
    .eq('album_id', album.id)
    .order('created_at', { ascending: true });

  if (memoriesError) {
    console.error('[public-album] Memories fetch error:', memoriesError);
    return NextResponse.json(
      { error: { code: 'INTERNAL_ERROR', message: 'Failed to load album content' } },
      { status: 500 }
    );
  }

  // 5. Génération des URLs signées pour les médias (durée 1h)
  const memoriesWithSignedUrls = await Promise.all(
    (memories ?? []).map(async ({ memory }) => {
      if (!memory || !memory.media_url) return memory;

      // media_url contient le storage_key, pas l'URL publique
      const { data: signedUrlData } = await adminClient.storage
        .from('travel-memories')
        .createSignedUrl(memory.media_url, 3600);

      return {
        ...memory,
        media_url: signedUrlData?.signedUrl ?? null,
      };
    })
  );

  // 6. Réponse publique — projection stricte, aucune donnée sensible
  const publicAlbum = {
    title: album.title,
    description: album.description,
    trip: {
      name: album.trip?.name ?? null,
      destination: album.trip?.destination_label ?? null,
    },
    memories: memoriesWithSignedUrls,
    memory_count: memoriesWithSignedUrls.length,
  };

  // Cache HTTP côté CDN : 5 minutes (album public mais contenu peut changer)
  return NextResponse.json(
    { data: publicAlbum },
    {
      status: 200,
      headers: {
        'Cache-Control': 'public, max-age=300, stale-while-revalidate=60',
      },
    }
  );
}
```

---

### 4.3 Invitation d'un participant (`POST /api/trips/:tripId/participants/invite`)

Route sensible : gestion de tokens d'invitation, contrôle des rôles, envoi de notification.

```typescript
// src/app/api/trips/[tripId]/participants/invite/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/client';
import { requireAuth } from '@/lib/middleware/auth';
import { requireTripRole } from '@/lib/middleware/tripRole';
import { rateLimit } from '@/lib/middleware/rateLimit';
import { sendInvitationEmail } from '@/lib/services/email';
import { nanoid } from 'nanoid';
import { z } from 'zod';

const InviteSchema = z.object({
  email: z.string().email('Invalid email address'),
  role: z.enum(['viewer', 'editor']).default('viewer'),
});

const MAX_PARTICIPANTS = 20;
const INVITATION_TTL_DAYS = 7;

export async function POST(
  request: NextRequest,
  { params }: { params: { tripId: string } }
) {
  // 1. Rate limiting — 10 invitations / heure par utilisateur
  const rateLimitResult = await rateLimit(request, {
    key: 'trip-invite',
    limit: 10,
    window: 3600,
  });
  if (!rateLimitResult.success) {
    return NextResponse.json(
      { error: { code: 'RATE_LIMIT_EXCEEDED', message: 'Too many invitations sent' } },
      { status: 429 }
    );
  }

  // 2. Auth + rôle admin requis pour inviter
  const supabase = createClient(request);
  const { user, error: authError } = await requireAuth(supabase);
  if (authError || !user) {
    return NextResponse.json(
      { error: { code: 'UNAUTHORIZED', message: 'Authentication required' } },
      { status: 401 }
    );
  }

  const { error: roleError } = await requireTripRole(
    supabase,
    params.tripId,
    user.id,
    ['admin']
  );
  if (roleError) {
    return NextResponse.json(
      { error: { code: 'FORBIDDEN', message: 'Only trip admins can invite participants' } },
      { status: 403 }
    );
  }

  // 3. Validation du body
  let body: unknown;
  try {
    body = await request.json();
  } catch {
    return NextResponse.json(
      { error: { code: 'INVALID_BODY', message: 'Expected JSON body' } },
      { status: 400 }
    );
  }

  const parsed = InviteSchema.safeParse(body);
  if (!parsed.success) {
    return NextResponse.json(
      { error: { code: 'VALIDATION_ERROR', message: parsed.error.errors[0].message } },
      { status: 422 }
    );
  }

  const { email, role } = parsed.data;

  // 4. Vérification limite participants
  const { count: participantCount } = await supabase
    .from('trip_participants')
    .select('id', { count: 'exact', head: true })
    .eq('trip_id', params.tripId);

  if ((participantCount ?? 0) >= MAX_PARTICIPANTS) {
    return NextResponse.json(
      { error: { code: 'PARTICIPANT_LIMIT_REACHED', message: `Maximum ${MAX_PARTICIPANTS} participants per trip` } },
      { status: 422 }
    );
  }

  // 5. Vérification si l'email est déjà participant ou a déjà une invitation pending
  const { data: existingUser } = await supabase
    .from('users')
    .select('id')
    .eq('email', email)
    .single();

  if (existingUser) {
    const { data: alreadyParticipant } = await supabase
      .from('trip_participants')
      .select('id')
      .eq('trip_id', params.tripId)
      .eq('user_id', existingUser.id)
      .single();

    if (alreadyParticipant) {
      return NextResponse.json(
        { error: { code: 'ALREADY_PARTICIPANT', message: 'User is already a participant of this trip' } },
        { status: 409 }
      );
    }
  }

  // Vérification invitation pending existante (évite les doublons)
  const { data: pendingInvitation } = await supabase
    .from('trip_invitations')
    .select('id')
    .eq('trip_id', params.tripId)
    .eq('invited_email', email)
    .eq('status', 'pending')
    .single();

  if (pendingInvitation) {
    return NextResponse.json(
      { error: { code: 'INVITATION_ALREADY_SENT', message: 'A pending invitation already exists for this email' } },
      { status: 409 }
    );
  }

  // 6. Création du token et de l'invitation
  const invitationToken = nanoid(32);
  const expiresAt = new Date();
  expiresAt.setDate(expiresAt.getDate() + INVITATION_TTL_DAYS);

  const { data: invitation, error: insertError } = await supabase
    .from('trip_invitations')
    .insert({
      trip_id: params.tripId,
      invited_by: user.id,
      invited_email: email,
      role,
      token: invitationToken,
      status: 'pending',
      expires_at: expiresAt.toISOString(),
    })
    .select()
    .single();

  if (insertError || !invitation) {
    console.error('[invite] DB error:', insertError);
    return NextResponse.json(
      { error: { code: 'DATABASE_ERROR', message: 'Failed to create invitation' } },
      { status: 500 }
    );
  }

  // 7. Récupération des infos du voyage pour l'email
  const { data: trip } = await supabase
    .from('trips')
    .select('name, destination_label')
    .eq('id', params.tripId)
    .single();

  // 8. Envoi email d'invitation (non bloquant : on ne fait pas échouer si l'email rate)
  try {
    await sendInvitationEmail({
      toEmail: email,
      inviterName: user.user_metadata?.full_name ?? user.email,
      tripName: trip?.name ?? 'a trip',
      destination: trip?.destination_label ?? '',
      invitationToken,
      expiresAt,
      role,
    });
  } catch (emailError) {
    // L'invitation est créée mais l'email a échoué — on log, on n'échoue pas la requête
    console.error('[invite] Email send failed (non-blocking):', emailError);
    // TODO: ajouter à une queue de retry
  }

  return NextResponse.json(
    {
      data: {
        id: invitation.id,
        invited_email: invitation.invited_email,
        role: invitation.role,
        status: invitation.status,
        expires_at: invitation.expires_at,
      },
    },
    { status: 201 }
  );
}
```

---

### 4.4 Proxy traduction (`POST /api/tools/translate`)

Route proxy vers DeepL. Centralise la clé API, valide les langues, gère le cache.

```typescript
// src/app/api/tools/translate/route.ts

import { NextRequest, NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/client';
import { requireAuth } from '@/lib/middleware/auth';
import { rateLimit } from '@/lib/middleware/rateLimit';
import { getRedisClient } from '@/lib/redis';
import { z } from 'zod';

// Liste des codes langue DeepL supportés (subset — compléter selon besoins)
const SUPPORTED_LANGUAGES = [
  'AR', 'BG', 'CS', 'DA', 'DE', 'EL', 'EN', 'ES', 'ET', 'FI',
  'FR', 'HU', 'ID', 'IT', 'JA', 'KO', 'LT', 'LV', 'NB', 'NL',
  'PL', 'PT',

— BACKEND ENGINEER

# Livrable Backend Engineer — Travel Companion App
## MVP B2C · Next.js API Routes · Supabase · Anthropic

---

## 1. SCHÉMA DE BASE DE DONNÉES COMPLET

### 1.1 Vue d'ensemble des tables

```
users                    ← géré par Supabase Auth + extension profil
trips                    ← entité centrale du domaine
trip_participants        ← relation utilisateur ↔ voyage + rôle
trip_invitations         ← tokens d'invitation par email
itinerary_steps          ← étapes d'un voyage (vol, hébergement, activité...)
documents                ← fichiers associés à un voyage ou une étape
memories                 ← souvenirs (photo + note + lieu)
memory_albums            ← albums de souvenirs partageables
album_memories           ← relation album ↔ souvenir (M2M)
currency_rates_cache     ← cache taux de change (évite appels redondants)
user_preferences         ← préférences utilisateur (devise, langue, notifs)
push_tokens              ← tokens FCM/APNs pour notifications push
scheduled_notifications  ← rappels planifiés sur les étapes
```

---

### 1.2 DDL complet — migrations SQL

```sql
-- ============================================================
-- EXTENSION : UUID génération
-- ============================================================
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm"; -- recherche texte

-- ============================================================
-- TABLE : user_profiles (extension de auth.users Supabase)
-- ============================================================
CREATE TABLE public.user_profiles (
  id              UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  full_name       TEXT,
  avatar_url      TEXT,
  email           TEXT NOT NULL, -- dénormalisé pour lisibilité
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ============================================================
-- TABLE : user_preferences
-- ============================================================
CREATE TABLE public.user_preferences (
  id                    UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id               UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  default_currency      CHAR(3) NOT NULL DEFAULT 'EUR', -- ISO 4217
  default_language      VARCHAR(10) NOT NULL DEFAULT 'fr', -- BCP 47
  notifications_enabled BOOLEAN NOT NULL DEFAULT TRUE,
  reminder_lead_time_h  INTEGER NOT NULL DEFAULT 24, -- heures avant étape
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(user_id)
);

-- ============================================================
-- TABLE : trips
-- ============================================================
CREATE TABLE public.trips (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  owner_id          UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  name              TEXT NOT NULL CHECK(length(name) BETWEEN 1 AND 120),
  destination_label TEXT NOT NULL, -- label lisible ex: "Tokyo, Japon"
  destination_lat   DECIMAL(9,6),
  destination_lng   DECIMAL(9,6),
  country_code      CHAR(2),       -- ISO 3166-1 alpha-2
  currency_code     CHAR(3),       -- devise locale destination
  language_code     VARCHAR(10),   -- langue locale destination
  timezone          TEXT,          -- ex: "Asia/Tokyo"
  start_date        DATE NOT NULL,
  end_date          DATE NOT NULL,
  status            TEXT NOT NULL DEFAULT 'upcoming'
                    CHECK(status IN ('upcoming', 'ongoing', 'past', 'archived')),
  cover_image_url   TEXT,
  notes             TEXT,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  CONSTRAINT dates_coherent CHECK(end_date >= start_date)
);

CREATE INDEX idx_trips_owner_id ON public.trips(owner_id);
CREATE INDEX idx_trips_status   ON public.trips(status);
CREATE INDEX idx_trips_dates    ON public.trips(start_date, end_date);

-- ============================================================
-- TABLE : trip_participants
-- ============================================================
CREATE TABLE public.trip_participants (
  id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id    UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  user_id    UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  role       TEXT NOT NULL DEFAULT 'viewer'
             CHECK(role IN ('admin', 'editor', 'viewer')),
  joined_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(trip_id, user_id)
);

CREATE INDEX idx_trip_participants_trip_id ON public.trip_participants(trip_id);
CREATE INDEX idx_trip_participants_user_id ON public.trip_participants(user_id);

-- ============================================================
-- TABLE : trip_invitations
-- ============================================================
CREATE TABLE public.trip_invitations (
  id             UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id        UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  invited_by     UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  invited_email  TEXT NOT NULL,
  role           TEXT NOT NULL DEFAULT 'viewer'
                 CHECK(role IN ('editor', 'viewer')),
  token          TEXT NOT NULL UNIQUE, -- nanoid(32)
  status         TEXT NOT NULL DEFAULT 'pending'
                 CHECK(status IN ('pending', 'accepted', 'declined', 'expired')),
  expires_at     TIMESTAMPTZ NOT NULL,
  accepted_at    TIMESTAMPTZ,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_trip_invitations_token    ON public.trip_invitations(token);
CREATE INDEX idx_trip_invitations_email    ON public.trip_invitations(invited_email);
CREATE INDEX idx_trip_invitations_trip_id  ON public.trip_invitations(trip_id);

-- ============================================================
-- TABLE : itinerary_steps
-- ============================================================
CREATE TABLE public.itinerary_steps (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id         UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  step_date       DATE NOT NULL,
  step_order      INTEGER NOT NULL DEFAULT 0, -- ordre dans la journée
  type            TEXT NOT NULL DEFAULT 'activity'
                  CHECK(type IN ('flight', 'accommodation', 'activity', 'transport', 'restaurant', 'other')),
  title           TEXT NOT NULL CHECK(length(title) BETWEEN 1 AND 200),
  location_label  TEXT,
  location_lat    DECIMAL(9,6),
  location_lng    DECIMAL(9,6),
  start_time      TIME,               -- heure locale de l'étape
  end_time        TIME,
  timezone        TEXT,               -- fuseau horaire de l'étape
  notes           TEXT,
  confirmation_ref TEXT,              -- numéro de réservation
  is_confirmed    BOOLEAN DEFAULT FALSE,
  created_by      UUID NOT NULL REFERENCES auth.users(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_steps_trip_id   ON public.itinerary_steps(trip_id);
CREATE INDEX idx_steps_step_date ON public.itinerary_steps(trip_id, step_date);

-- ============================================================
-- TABLE : documents
-- ============================================================
CREATE TABLE public.documents (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id         UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  step_id         UUID REFERENCES public.itinerary_steps(id) ON DELETE SET NULL,
  uploaded_by     UUID NOT NULL REFERENCES auth.users(id),
  category        TEXT NOT NULL DEFAULT 'other'
                  CHECK(category IN ('flight', 'accommodation', 'activity', 'transport', 'identity', 'insurance', 'other')),
  filename        TEXT NOT NULL,
  storage_key     TEXT NOT NULL UNIQUE, -- chemin dans Supabase Storage
  mime_type       TEXT NOT NULL,
  size_bytes      BIGINT NOT NULL CHECK(size_bytes <= 20971520), -- 20 Mo max
  is_cached       BOOLEAN DEFAULT FALSE, -- indique si disponible offline
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_documents_trip_id  ON public.documents(trip_id);
CREATE INDEX idx_documents_step_id  ON public.documents(step_id);
CREATE INDEX idx_documents_category ON public.documents(trip_id, category);

-- ============================================================
-- TABLE : memories
-- ============================================================
CREATE TABLE public.memories (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id       UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  created_by    UUID NOT NULL REFERENCES auth.users(id),
  type          TEXT NOT NULL DEFAULT 'photo'
                CHECK(type IN ('photo', 'note', 'photo_note')),
  media_url     TEXT,      -- storage_key dans Supabase Storage
  note          TEXT CHECK(length(note) <= 2000),
  location_label TEXT,
  location_lat  DECIMAL(9,6),
  location_lng  DECIMAL(9,6),
  taken_at      TIMESTAMPTZ, -- date réelle de la photo/note
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_memories_trip_id    ON public.memories(trip_id);
CREATE INDEX idx_memories_created_by ON public.memories(created_by);
CREATE INDEX idx_memories_taken_at   ON public.memories(trip_id, taken_at DESC);

-- ============================================================
-- TABLE : memory_albums
-- ============================================================
CREATE TABLE public.memory_albums (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  trip_id       UUID NOT NULL REFERENCES public.trips(id) ON DELETE CASCADE,
  created_by    UUID NOT NULL REFERENCES auth.users(id),
  title         TEXT NOT NULL CHECK(length(title) BETWEEN 1 AND 100),
  description   TEXT CHECK(length(description) <= 500),
  share_token   TEXT UNIQUE,          -- null = album privé
  share_enabled BOOLEAN DEFAULT FALSE,
  share_expires_at TIMESTAMPTZ,       -- null = pas d'expiration
  view_count    INTEGER DEFAULT 0,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_albums_trip_id     ON public.memory_albums(trip_id);
CREATE INDEX idx_albums_share_token ON public.memory_albums(share_token);

-- ============================================================
-- TABLE : album_memories (M2M)
-- ============================================================
CREATE TABLE public.album_memories (
  album_id    UUID NOT NULL REFERENCES public.memory_albums(id) ON DELETE CASCADE,
  memory_id   UUID NOT NULL REFERENCES public.memories(id) ON DELETE CASCADE,
  added_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  sort_order  INTEGER DEFAULT 0,
  PRIMARY KEY (album_id, memory_id)
);

-- ============================================================
-- TABLE : currency_rates_cache
-- ============================================================
CREATE TABLE public.currency_rates_cache (
  base_currency CHAR(3) NOT NULL,
  rates         JSONB NOT NULL,       -- { "USD": 1.08, "GBP": 0.86, ... }
  fetched_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (base_currency)
);

-- ============================================================
-- TABLE : push_tokens
-- ============================================================
CREATE TABLE public.push_tokens (
  id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id     UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  token       TEXT NOT NULL UNIQUE,
  platform    TEXT NOT NULL CHECK(platform IN ('ios', 'android', 'web')),
  is_active   BOOLEAN DEFAULT TRUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_push_tokens_user_id ON public.push_tokens(user_id);

-- ============================================================
-- TABLE : scheduled_notifications
-- ============================================================
CREATE TABLE public.scheduled_notifications (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  step_id         UUID NOT NULL REFERENCES public.itinerary_steps(id) ON DELETE CASCADE,
  user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  scheduled_for   TIMESTAMPTZ NOT NULL,
  lead_time_h     INTEGER NOT NULL,   -- ex: 24, 2
  type            TEXT NOT NULL DEFAULT 'step_reminder',
  payload         JSONB,              -- données pour la notification
  status          TEXT NOT NULL DEFAULT 'pending'
                  CHECK(status IN ('pending', 'sent', 'failed', 'cancelled')),
  sent_at         TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifs_scheduled_for ON public.scheduled_notifications(scheduled_for)
  WHERE status = 'pending';
CREATE INDEX idx_notifs_step_id ON public.scheduled_notifications(step_id);
```

---

### 1.3 Row Level Security (RLS)

```sql
-- ============================================================
-- RLS — Activation sur toutes les tables
-- ============================================================
ALTER TABLE public.user_profiles       ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.user_preferences    ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.trips               ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.trip_participants   ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.trip_invitations    ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.itinerary_steps     ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.documents           ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.memories            ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.memory_albums       ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.album_memories      ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.push_tokens         ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.scheduled_notifications ENABLE ROW LEVEL SECURITY;

-- ============================================================
-- FONCTION HELPER : vérifier le rôle d'un user sur un trip
-- ============================================================
CREATE OR REPLACE FUNCTION public.get_trip_role(p_trip_id UUID, p_user_id UUID)
RETURNS TEXT AS $$
  SELECT role FROM public.trip_participants
  WHERE trip_id = p_trip_id AND user_id = p_user_id
  LIMIT 1;
$$ LANGUAGE sql STABLE SECURITY DEFINER;

-- ============================================================
-- RLS : user_profiles
-- ============================================================
CREATE POLICY "users_own_profile_select"
  ON public.user_profiles FOR SELECT
  USING (id = auth.uid());

CREATE POLICY "users_own_profile_update"
  ON public.user_profiles FOR UPDATE
  USING (id = auth.uid());

-- Les participants d'un même voyage peuvent voir les profils des autres
CREATE POLICY "trip_participants_can_see_profiles"
  ON public.user_profiles FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM public.trip_participants tp1
      JOIN public.trip_participants tp2 ON tp1.trip_id = tp2.trip_id
      WHERE tp1.user_id = auth.uid()
        AND tp2.user_id = public.user_profiles.id
    )
  );

-- ============================================================
-- RLS : trips
-- ============================================================
CREATE POLICY "trips_participant_select"
  ON public.trips FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM public.trip_participants
      WHERE trip_id = id AND user_id = auth.uid()
    )
  );

CREATE POLICY "trips_owner_insert"
  ON public.trips FOR INSERT
  WITH CHECK (owner_id = auth.uid());

CREATE POLICY