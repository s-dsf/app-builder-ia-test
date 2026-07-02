# frontend_engineer

**Projet :** app voyage 3
**Mis à jour :** 01/07/2026

---

— FRONTEND ENGINEER

# Livrable Frontend Engineer — Travel Companion App
## MVP B2C · Next.js 14 · Tailwind CSS · React Native (alignement tokens)

---

> **Note de cadrage** : L'architecture retenue est **React Native** pour le mobile (défini par l'Architecte et le Tech Lead). Ce livrable Frontend couvre l'**interface web companion** (tableau de bord B2C, onboarding web, partage de souvenirs public) en Next.js 14, et fournit les **tokens de design system, patterns de composants et conventions** directement réutilisables par le développeur React Native pour l'app mobile. Les deux surfaces partagent la même logique de composants et les mêmes patterns d'état.

---

## 1. ARCHITECTURE FRONTEND

### 1.1 Structure des dossiers — Next.js 14 (App Router)

```
src/
├── app/                              # App Router Next.js 14
│   ├── (auth)/                       # Route group — pages non protégées
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── register/
│   │   │   └── page.tsx
│   │   └── layout.tsx                # Layout minimaliste auth
│   ├── (app)/                        # Route group — pages protégées
│   │   ├── layout.tsx                # Layout principal avec nav
│   │   ├── dashboard/
│   │   │   └── page.tsx              # Tableau de bord voyages
│   │   ├── trip/
│   │   │   ├── new/
│   │   │   │   └── page.tsx          # Création d'un voyage
│   │   │   └── [tripId]/
│   │   │       ├── layout.tsx        # Layout voyage avec tabs
│   │   │       ├── page.tsx          # Redirect → itinerary
│   │   │       ├── itinerary/
│   │   │       │   └── page.tsx      # Vue itinéraire jour/j
│   │   │       ├── documents/
│   │   │       │   └── page.tsx      # Documents catégorisés
│   │   │       ├── memories/
│   │   │       │   └── page.tsx      # Souvenirs & partage
│   │   │       └── participants/
│   │   │           └── page.tsx      # Gestion participants
│   │   ├── tools/
│   │   │   └── page.tsx              # Hub outils utilitaires
│   │   └── settings/
│   │       └── page.tsx              # Paramètres utilisateur
│   ├── share/
│   │   └── [albumId]/
│   │       └── page.tsx              # Page publique album souvenirs
│   ├── globals.css
│   └── layout.tsx                    # Root layout
│
├── components/
│   ├── ui/                           # Composants atomiques (Design System)
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   └── Button.stories.tsx
│   │   ├── Input/
│   │   ├── Card/
│   │   ├── Badge/
│   │   ├── Avatar/
│   │   ├── Modal/
│   │   ├── BottomSheet/
│   │   ├── Skeleton/
│   │   └── OfflineBanner/
│   │
│   ├── layout/                       # Composants de structure
│   │   ├── AppShell.tsx              # Navigation principale + layout
│   │   ├── TripTabBar.tsx            # Tabs dans la vue voyage
│   │   ├── ToolsDrawer.tsx           # Tiroir outils persistant
│   │   └── MobileBottomNav.tsx       # Navigation bas mobile
│   │
│   ├── trip/                         # Composants domaine Voyage
│   │   ├── TripCard.tsx              # Carte voyage (dashboard)
│   │   ├── TripEmptyState.tsx        # État vide guidé
│   │   ├── TripCreationForm.tsx      # Formulaire création voyage
│   │   ├── ItineraryDayView.tsx      # Vue journée itinéraire
│   │   ├── StepCard.tsx              # Carte étape individuelle
│   │   ├── StepDetail.tsx            # Détail d'une étape
│   │   └── StepForm.tsx              # Formulaire ajout/édition étape
│   │
│   ├── documents/                    # Composants domaine Documents
│   │   ├── DocumentList.tsx          # Liste catégorisée
│   │   ├── DocumentCard.tsx          # Carte document individuel
│   │   ├── DocumentUploader.tsx      # Zone upload (drag & drop + mobile)
│   │   └── DocumentViewer.tsx        # Visionneuse inline
│   │
│   ├── tools/                        # Composants domaine Outils
│   │   ├── TranslatorWidget.tsx      # Traducteur
│   │   ├── CurrencyConverter.tsx     # Convertisseur devises
│   │   └── TimezoneDisplay.tsx       # Horloge / fuseaux
│   │
│   └── memories/                     # Composants domaine Souvenirs
│       ├── MemoryGallery.tsx          # Galerie photos/notes
│       ├── MemoryCard.tsx             # Carte souvenir individuel
│       ├── MemoryAddForm.tsx          # Formulaire ajout souvenir
│       └── SharedAlbumView.tsx        # Vue publique album
│
├── hooks/                            # Hooks custom
│   ├── useActiveTrip.ts              # Voyage actif en contexte global
│   ├── useOfflineStatus.ts           # Détection connexion réseau
│   ├── useTripItinerary.ts           # Données itinéraire + cache
│   ├── useDocuments.ts               # Gestion documents
│   ├── useTranslator.ts              # Appels traducteur
│   └── useCurrencyConverter.ts       # Conversion devises
│
├── store/                            # État global (Zustand)
│   ├── authStore.ts                  # Session utilisateur
│   ├── tripStore.ts                  # Voyage actif + liste voyages
│   ├── uiStore.ts                    # État UI (modals, drawers...)
│   └── offlineStore.ts               # File sync offline
│
├── lib/                              # Utilitaires et configuration
│   ├── api/
│   │   ├── client.ts                 # Axios instance + intercepteurs
│   │   ├── trips.ts                  # Endpoints voyages
│   │   ├── documents.ts              # Endpoints documents
│   │   ├── tools.ts                  # Endpoints outils
│   │   └── memories.ts               # Endpoints souvenirs
│   ├── cache/
│   │   └── offlineCache.ts           # Stratégie cache offline
│   └── utils/
│       ├── dates.ts                  # Formatage dates, fuseaux
│       ├── fileSize.ts               # Validation taille fichiers
│       └── tripStatus.ts             # Calcul statut voyage
│
├── styles/
│   └── tokens.css                    # Variables CSS design system
│
└── types/
    ├── trip.ts                       # Types domaine voyage
    ├── document.ts                   # Types documents
    ├── tools.ts                      # Types outils
    └── user.ts                       # Types utilisateur
```

---

### 1.2 Conventions de développement

| Convention | Règle |
|---|---|
| **Composants** | PascalCase, un composant par fichier, colocalisé avec ses types |
| **Hooks** | Préfixe `use`, retournent `{ data, isLoading, error, ...actions }` |
| **Pages** | Légères — orchestrent des composants, ne contiennent pas de logique |
| **Imports** | Alias `@/` pour `src/` — jamais de chemins relatifs profonds |
| **Types** | Partagés dans `/types`, jamais inline dans les composants |
| **API calls** | Exclusivement dans les hooks — jamais dans les composants |
| **Offline first** | Tout composant vérifie `useOfflineStatus` avant appel réseau |

---

## 2. DESIGN SYSTEM — TOKENS

### 2.1 tokens.css

```css
/* src/styles/tokens.css */
:root {
  /* ─── PALETTE PRINCIPALE ─── */
  --color-primary-50:  #EFF8FF;
  --color-primary-100: #DBEEFE;
  --color-primary-200: #BFE0FD;
  --color-primary-300: #93CBFB;
  --color-primary-400: #60AFF7;
  --color-primary-500: #3B91F3;   /* Brand principal */
  --color-primary-600: #2573E1;
  --color-primary-700: #1D5CC8;
  --color-primary-800: #1E4BA3;
  --color-primary-900: #1E3F81;

  /* ─── PALETTE SECONDAIRE (chaud, voyage) ─── */
  --color-sand-50:  #FEFCE8;
  --color-sand-100: #FEF9C3;
  --color-sand-200: #FEF08A;
  --color-sand-300: #FDE047;
  --color-sand-400: #FACC15;
  --color-sand-500: #EAB308;   /* Accent chaud */
  --color-sand-600: #CA8A04;
  --color-sand-700: #A16207;

  /* ─── SÉMANTIQUE ─── */
  --color-success: #22C55E;
  --color-warning: #F97316;
  --color-error:   #EF4444;
  --color-info:    #3B91F3;

  /* ─── NEUTRALS ─── */
  --color-neutral-0:   #FFFFFF;
  --color-neutral-50:  #F8FAFC;
  --color-neutral-100: #F1F5F9;
  --color-neutral-200: #E2E8F0;
  --color-neutral-300: #CBD5E1;
  --color-neutral-400: #94A3B8;
  --color-neutral-500: #64748B;
  --color-neutral-600: #475569;
  --color-neutral-700: #334155;
  --color-neutral-800: #1E293B;
  --color-neutral-900: #0F172A;

  /* ─── TYPOGRAPHIE ─── */
  --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  --font-display: 'Sora', var(--font-sans);

  --text-xs:   0.75rem;    /* 12px */
  --text-sm:   0.875rem;   /* 14px */
  --text-base: 1rem;       /* 16px */
  --text-lg:   1.125rem;   /* 18px */
  --text-xl:   1.25rem;    /* 20px */
  --text-2xl:  1.5rem;     /* 24px */
  --text-3xl:  1.875rem;   /* 30px */

  /* ─── ESPACEMENT ─── */
  --space-1:  0.25rem;   /* 4px */
  --space-2:  0.5rem;    /* 8px */
  --space-3:  0.75rem;   /* 12px */
  --space-4:  1rem;      /* 16px */
  --space-5:  1.25rem;   /* 20px */
  --space-6:  1.5rem;    /* 24px */
  --space-8:  2rem;      /* 32px */
  --space-10: 2.5rem;    /* 40px */
  --space-12: 3rem;      /* 48px */
  --space-16: 4rem;      /* 64px */

  /* ─── BORDER RADIUS ─── */
  --radius-sm:   0.375rem;  /* 6px */
  --radius-md:   0.75rem;   /* 12px */
  --radius-lg:   1rem;      /* 16px */
  --radius-xl:   1.5rem;    /* 24px */
  --radius-full: 9999px;

  /* ─── OMBRES ─── */
  --shadow-sm:  0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md:  0 4px 6px -1px rgb(0 0 0 / 0.1);
  --shadow-lg:  0 10px 15px -3px rgb(0 0 0 / 0.1);
  --shadow-card: 0 2px 8px 0 rgb(15 23 42 / 0.08);

  /* ─── TRANSITIONS ─── */
  --transition-fast:   150ms ease;
  --transition-base:   250ms ease;
  --transition-slow:   400ms ease;

  /* ─── OFFLINE ─── */
  --offline-overlay: rgb(15 23 42 / 0.04);
  --offline-stripe: repeating-linear-gradient(
    -45deg,
    transparent,
    transparent 4px,
    var(--offline-overlay) 4px,
    var(--offline-overlay) 8px
  );
}
```

### 2.2 Tailwind config (extrait)

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss'

const config: Config = {
  content: ['./src/**/*.{js,ts,jsx,tsx,mdx}'],
  theme: {
    extend: {
      colors: {
        primary: {
          50:  'var(--color-primary-50)',
          500: 'var(--color-primary-500)',
          600: 'var(--color-primary-600)',
          700: 'var(--color-primary-700)',
        },
        sand: {
          300: 'var(--color-sand-300)',
          500: 'var(--color-sand-500)',
        },
        neutral: {
          50:  'var(--color-neutral-50)',
          100: 'var(--color-neutral-100)',
          200: 'var(--color-neutral-200)',
          400: 'var(--color-neutral-400)',
          600: 'var(--color-neutral-600)',
          800: 'var(--color-neutral-800)',
          900: 'var(--color-neutral-900)',
        }
      },
      fontFamily: {
        sans:    ['Inter', 'system-ui', 'sans-serif'],
        display: ['Sora', 'Inter', 'sans-serif'],
      },
      borderRadius: {
        'card': '1rem',
        'pill': '9999px',
      },
      boxShadow: {
        'card': '0 2px 8px 0 rgb(15 23 42 / 0.08)',
        'card-hover': '0 8px 24px 0 rgb(15 23 42 / 0.12)',
      },
      screens: {
        'xs': '390px',   // iPhone SE / petits mobiles
      }
    },
  },
  plugins: [],
}

export default config
```

---

## 3.

— FRONTEND ENGINEER

## 3. PAGES — INVENTAIRE ET RESPONSABILITÉS

| Page | Route | Responsabilité | Composants clés |
|---|---|---|---|
| **Onboarding** | `/onboarding` | 3 slides valeur proposition + CTA | `OnboardingSlider`, `OnboardingSlide` |
| **Login** | `/login` | Connexion email/password + OAuth | `AuthForm`, `SocialAuthButtons` |
| **Register** | `/register` | Inscription email/password | `AuthForm`, `PasswordStrength` |
| **Dashboard** | `/` | Voyage actif + liste voyages + accès outils | `TripCard`, `TripList`, `ToolsQuickAccess` |
| **Trip Detail** | `/trips/[id]` | Hub central du voyage, tabs navigation | `TripHeader`, `TabNav` |
| **Itinerary** | `/trips/[id]/itinerary` | Vue chronologique jour par jour | `ItineraryDayView`, `StepCard` |
| **Step Detail** | `/trips/[id]/itinerary/[stepId]` | Détail complet d'une étape | `StepDetail`, `DocumentMiniList` |
| **Documents** | `/trips/[id]/documents` | Liste catégorisée + upload | `DocumentList`, `DocumentUploader` |
| **Memories** | `/trips/[id]/memories` | Galerie souvenirs + ajout | `MemoryGallery`, `MemoryAddForm` |
| **Tools** | `/tools` | Hub outils contextualisés | `TranslatorWidget`, `CurrencyConverter`, `TimezoneDisplay` |
| **Shared Album** | `/shared/[token]` | Vue publique album (non authentifié) | `SharedAlbumView` |
| **Settings** | `/settings` | Préférences, profil, déconnexion | `ProfileForm`, `NotificationPrefs` |
| **Trip Creation** | `/trips/new` | Formulaire création voyage | `TripCreationForm` |

---

## 4. COMPOSANTS CLÉS — CODE

### 4.1 `TripCard.tsx` — Carte voyage (Dashboard)

Le composant le plus affiché de l'app. Doit gérer 3 états (à venir / en cours / passé), l'indicateur offline et le contexte voyage actif.

```tsx
// src/components/trips/TripCard.tsx
'use client'

import { memo } from 'react'
import Link from 'next/link'
import { MapPin, Calendar, Users, WifiOff, ChevronRight } from 'lucide-react'
import { cn } from '@/lib/utils/cn'
import { useTripStatus } from '@/hooks/useTripStatus'
import { useOfflineStatus } from '@/hooks/useOfflineStatus'
import type { Trip } from '@/types/trip'

// ─── Types ────────────────────────────────────────────────────────────────────

interface TripCardProps {
  trip: Trip
  isActive?: boolean       // Voyage actuellement en cours affiché en hero
  variant?: 'hero' | 'compact'
  className?: string
}

// ─── Constantes ───────────────────────────────────────────────────────────────

const STATUS_CONFIG = {
  upcoming: {
    label: 'À venir',
    className: 'bg-primary-50 text-primary-700 border border-primary-200',
  },
  ongoing: {
    label: 'En cours',
    className: 'bg-green-50 text-green-700 border border-green-200',
  },
  past: {
    label: 'Passé',
    className: 'bg-neutral-100 text-neutral-500 border border-neutral-200',
  },
} as const

// ─── Composant ────────────────────────────────────────────────────────────────

export const TripCard = memo(function TripCard({
  trip,
  isActive = false,
  variant = 'compact',
  className,
}: TripCardProps) {
  const { status, daysUntilDeparture, daysRemaining } = useTripStatus(trip)
  const { isOffline } = useOfflineStatus()

  const statusConfig = STATUS_CONFIG[status]
  const isHero = variant === 'hero'

  // Formatage des dates
  const dateRange = formatDateRange(trip.startDate, trip.endDate)

  return (
    <Link
      href={`/trips/${trip.id}`}
      className={cn(
        // Base
        'group relative block rounded-card bg-white shadow-card',
        'transition-all duration-[var(--transition-base)]',
        'hover:shadow-card-hover hover:-translate-y-0.5',
        // Focus accessible
        'focus-visible:outline-none focus-visible:ring-2',
        'focus-visible:ring-primary-500 focus-visible:ring-offset-2',
        // Variantes
        isHero && 'overflow-hidden',
        // Voyage passé : légèrement atténué
        status === 'past' && 'opacity-75',
        className
      )}
      aria-label={`Voyage ${trip.name} — ${statusConfig.label} — ${dateRange}`}
    >
      {/* ── Image de couverture (hero uniquement) ── */}
      {isHero && trip.coverImageUrl && (
        <div className="relative h-40 w-full overflow-hidden">
          {/* eslint-disable-next-line @next/next/no-img-element */}
          <img
            src={trip.coverImageUrl}
            alt=""
            aria-hidden="true"
            className="h-full w-full object-cover transition-transform duration-500 group-hover:scale-105"
          />
          {/* Gradient overlay */}
          <div className="absolute inset-0 bg-gradient-to-t from-neutral-900/60 to-transparent" />
          
          {/* Badge offline sur l'image */}
          {isOffline && (
            <div className="absolute right-3 top-3 flex items-center gap-1.5 rounded-pill bg-neutral-900/70 px-2.5 py-1 backdrop-blur-sm">
              <WifiOff className="h-3 w-3 text-white" aria-hidden="true" />
              <span className="text-xs font-medium text-white">Hors ligne</span>
            </div>
          )}
        </div>
      )}

      {/* ── Corps de la carte ── */}
      <div className={cn('p-4', isHero && 'p-5')}>
        {/* Header : nom + statut */}
        <div className="flex items-start justify-between gap-3">
          <div className="flex-1 min-w-0">
            <h3
              className={cn(
                'font-display font-semibold text-neutral-900 truncate',
                isHero ? 'text-xl' : 'text-base'
              )}
            >
              {trip.name}
            </h3>
          </div>
          
          {/* Badge statut */}
          <span
            className={cn(
              'shrink-0 rounded-pill px-2.5 py-0.5 text-xs font-medium',
              statusConfig.className
            )}
          >
            {statusConfig.label}
          </span>
        </div>

        {/* Métadonnées */}
        <dl className="mt-3 space-y-1.5">
          {/* Destination */}
          <div className="flex items-center gap-2 text-neutral-600">
            <MapPin className="h-4 w-4 shrink-0 text-neutral-400" aria-hidden="true" />
            <dt className="sr-only">Destination</dt>
            <dd className="text-sm truncate">{trip.destination}</dd>
          </div>

          {/* Dates */}
          <div className="flex items-center gap-2 text-neutral-600">
            <Calendar className="h-4 w-4 shrink-0 text-neutral-400" aria-hidden="true" />
            <dt className="sr-only">Dates</dt>
            <dd className="text-sm">{dateRange}</dd>
          </div>

          {/* Participants (si > 1) */}
          {trip.participantCount > 1 && (
            <div className="flex items-center gap-2 text-neutral-600">
              <Users className="h-4 w-4 shrink-0 text-neutral-400" aria-hidden="true" />
              <dt className="sr-only">Participants</dt>
              <dd className="text-sm">
                {trip.participantCount} voyageur{trip.participantCount > 2 ? 's' : ''}
              </dd>
            </div>
          )}
        </dl>

        {/* Indicateur contextuel selon statut */}
        {status === 'upcoming' && daysUntilDeparture !== null && (
          <p className="mt-3 text-sm font-medium text-primary-600">
            {daysUntilDeparture === 0
              ? "C'est aujourd'hui ! 🎉"
              : daysUntilDeparture === 1
              ? 'Départ demain'
              : `Départ dans ${daysUntilDeparture} jours`}
          </p>
        )}

        {status === 'ongoing' && daysRemaining !== null && (
          <div className="mt-3 flex items-center justify-between">
            <p className="text-sm font-medium text-green-600">
              {daysRemaining === 0 ? 'Dernier jour !' : `Encore ${daysRemaining} jour${daysRemaining > 1 ? 's' : ''}`}
            </p>
            {/* Barre de progression */}
            <TripProgressBar trip={trip} className="w-24" />
          </div>
        )}

        {/* CTA chevron */}
        <div
          className="absolute right-4 top-1/2 -translate-y-1/2 text-neutral-300 transition-transform duration-[var(--transition-fast)] group-hover:translate-x-0.5 group-hover:text-primary-500"
          aria-hidden="true"
        >
          <ChevronRight className="h-5 w-5" />
        </div>
      </div>

      {/* Indicateur offline en bas de carte (variant compact) */}
      {isOffline && variant === 'compact' && (
        <div className="flex items-center gap-1.5 border-t border-neutral-100 px-4 py-2">
          <WifiOff className="h-3 w-3 text-neutral-400" aria-hidden="true" />
          <span className="text-xs text-neutral-400">Disponible hors ligne</span>
        </div>
      )}
    </Link>
  )
})

// ─── Sous-composant : barre de progression ────────────────────────────────────

function TripProgressBar({ trip, className }: { trip: Trip; className?: string }) {
  const totalDays = Math.max(
    1,
    Math.round(
      (new Date(trip.endDate).getTime() - new Date(trip.startDate).getTime()) /
        (1000 * 60 * 60 * 24)
    )
  )
  const elapsed = Math.round(
    (Date.now() - new Date(trip.startDate).getTime()) / (1000 * 60 * 60 * 24)
  )
  const progress = Math.min(100, Math.max(0, (elapsed / totalDays) * 100))

  return (
    <div
      className={cn('h-1.5 overflow-hidden rounded-full bg-neutral-100', className)}
      role="progressbar"
      aria-valuenow={Math.round(progress)}
      aria-valuemin={0}
      aria-valuemax={100}
      aria-label="Progression du voyage"
    >
      <div
        className="h-full rounded-full bg-green-400 transition-all duration-[var(--transition-slow)]"
        style={{ width: `${progress}%` }}
      />
    </div>
  )
}

// ─── Utilitaire ───────────────────────────────────────────────────────────────

function formatDateRange(start: string, end: string): string {
  const startDate = new Date(start)
  const endDate = new Date(end)
  const opts: Intl.DateTimeFormatOptions = { day: 'numeric', month: 'short' }
  
  if (startDate.getFullYear() !== endDate.getFullYear()) {
    return `${startDate.toLocaleDateString('fr-FR', { ...opts, year: 'numeric' })} → ${endDate.toLocaleDateString('fr-FR', { ...opts, year: 'numeric' })}`
  }
  return `${startDate.toLocaleDateString('fr-FR', opts)} → ${endDate.toLocaleDateString('fr-FR', opts)} ${endDate.getFullYear()}`
}
```

---

### 4.2 `ItineraryDayView.tsx` — Vue journée itinéraire

Composant central de la consultation voyage. Gère l'affichage chronologique, la navigation entre jours et l'état offline.

```tsx
// src/components/trips/ItineraryDayView.tsx
'use client'

import { useState, useCallback, useId } from 'react'
import { ChevronLeft, ChevronRight, Plus, CloudOff } from 'lucide-react'
import { cn } from '@/lib/utils/cn'
import { StepCard } from './StepCard'
import { TripEmptyState } from './TripEmptyState'
import { useOfflineStatus } from '@/hooks/useOfflineStatus'
import type { ItineraryDay, TripStep } from '@/types/trip'

// ─── Types ────────────────────────────────────────────────────────────────────

interface ItineraryDayViewProps {
  days: ItineraryDay[]
  activeDayIndex?: number
  tripId: string
  canEdit?: boolean
  onAddStep?: (dayDate: string) => void
  onStepClick?: (step: TripStep) => void
  className?: string
}

// ─── Composant ────────────────────────────────────────────────────────────────

export function ItineraryDayView({
  days,
  activeDayIndex: initialDayIndex = 0,
  tripId,
  canEdit = false,
  onAddStep,
  onStepClick,
  className,
}: ItineraryDayViewProps) {
  const { isOffline } = useOfflineStatus()
  const [currentDayIndex, setCurrentDayIndex] = useState(
    // Auto-positionner sur aujourd'hui si pertinent
    findTodayIndex(days) ?? initialDayIndex
  )

  const headingId = useId()
  const currentDay = days[currentDayIndex]
  const hasPrev = currentDayIndex > 0
  const hasNext = currentDayIndex < days.length - 1

  const goToPrev = useCallback(() => {
    setCurrentDayIndex((i) => Math.max(0, i - 1))
  }, [])

  const goToNext = useCallback(() => {
    setCurrentDayIndex((i) => Math.min(days.length - 1, i + 1))
  }, [days.length])

  // ── Cas vide ──
  if (days.length === 0) {
    return (
      <TripEmptyState
        variant="itinerary"
        canEdit={canEdit}
        onAction={() => onAddStep?.('')}
      />
    )
  }

  return (
    <section
      className={cn('flex flex-col gap-4', className)}
      aria

— FRONTEND ENGINEER

*(continuation exacte depuis la dernière ligne produite)*

```tsx
      aria-labelledby={headingId}
    >
      {/* ── Navigation jours ── */}
      <div className="flex items-center justify-between gap-2">
        <button
          onClick={goToPrev}
          disabled={!hasPrev}
          className={cn(
            'flex h-9 w-9 items-center justify-center rounded-full border border-neutral-200 text-neutral-600',
            'transition-colors duration-[var(--transition-fast)]',
            'hover:border-primary-300 hover:text-primary-600',
            'disabled:cursor-not-allowed disabled:opacity-30',
            'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500'
          )}
          aria-label="Jour précédent"
        >
          <ChevronLeft className="h-4 w-4" aria-hidden="true" />
        </button>

        {/* Sélecteur de jour : chips scrollables */}
        <div
          className="flex flex-1 gap-1.5 overflow-x-auto scrollbar-none"
          role="tablist"
          aria-label="Jours du voyage"
        >
          {days.map((day, index) => {
            const isActive = index === currentDayIndex
            const isToday = isSameDay(new Date(day.date), new Date())
            return (
              <button
                key={day.date}
                role="tab"
                aria-selected={isActive}
                aria-controls={`day-panel-${index}`}
                onClick={() => setCurrentDayIndex(index)}
                className={cn(
                  'shrink-0 rounded-pill px-3 py-1.5 text-xs font-medium transition-colors duration-[var(--transition-fast)]',
                  'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500',
                  isActive
                    ? 'bg-primary-500 text-white shadow-sm'
                    : 'bg-neutral-100 text-neutral-600 hover:bg-neutral-200',
                  isToday && !isActive && 'ring-1 ring-primary-300'
                )}
              >
                <span className="sr-only">Jour </span>
                J{index + 1}
                {isToday && <span className="ml-1 text-[10px] opacity-80">•</span>}
              </button>
            )
          })}
        </div>

        <button
          onClick={goToNext}
          disabled={!hasNext}
          className={cn(
            'flex h-9 w-9 items-center justify-center rounded-full border border-neutral-200 text-neutral-600',
            'transition-colors duration-[var(--transition-fast)]',
            'hover:border-primary-300 hover:text-primary-600',
            'disabled:cursor-not-allowed disabled:opacity-30',
            'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500'
          )}
          aria-label="Jour suivant"
        >
          <ChevronRight className="h-4 w-4" aria-hidden="true" />
        </button>
      </div>

      {/* ── En-tête du jour courant ── */}
      <div className="flex items-center justify-between">
        <div>
          <h2
            id={headingId}
            className="font-display text-lg font-semibold text-neutral-900"
          >
            {formatDayHeading(currentDay.date, currentDayIndex)}
          </h2>
          {currentDay.location && (
            <p className="mt-0.5 text-sm text-neutral-500">{currentDay.location}</p>
          )}
        </div>

        {/* Indicateur offline discret */}
        {isOffline && (
          <div
            className="flex items-center gap-1 rounded-pill bg-neutral-100 px-2.5 py-1"
            title="Données en cache — lecture seule"
          >
            <CloudOff className="h-3 w-3 text-neutral-400" aria-hidden="true" />
            <span className="text-xs text-neutral-400">Cache</span>
          </div>
        )}
      </div>

      {/* ── Liste des étapes ── */}
      <div
        id={`day-panel-${currentDayIndex}`}
        role="tabpanel"
        aria-labelledby={headingId}
        className="flex flex-col gap-2"
      >
        {currentDay.steps.length === 0 ? (
          <div className="rounded-2xl border border-dashed border-neutral-200 px-6 py-10 text-center">
            <p className="text-sm text-neutral-400">Aucune étape pour ce jour</p>
            {canEdit && !isOffline && (
              <button
                onClick={() => onAddStep?.(currentDay.date)}
                className="mt-3 text-sm font-medium text-primary-500 hover:text-primary-600"
              >
                + Ajouter une étape
              </button>
            )}
          </div>
        ) : (
          <>
            {/* Timeline verticale */}
            <ol className="relative pl-6" aria-label="Étapes du jour">
              {/* Ligne verticale */}
              <div
                className="absolute left-2 top-2 h-[calc(100%-1rem)] w-px bg-neutral-100"
                aria-hidden="true"
              />

              {currentDay.steps.map((step, stepIndex) => (
                <li key={step.id} className="relative mb-3 last:mb-0">
                  {/* Point timeline */}
                  <div
                    className={cn(
                      'absolute -left-4 top-3.5 h-2.5 w-2.5 rounded-full border-2 border-white',
                      getStepDotColor(step.type)
                    )}
                    aria-hidden="true"
                  />
                  <StepCard
                    step={step}
                    onClick={() => onStepClick?.(step)}
                    isOffline={isOffline}
                  />
                </li>
              ))}
            </ol>

            {/* Bouton ajout en fin de liste */}
            {canEdit && !isOffline && (
              <button
                onClick={() => onAddStep?.(currentDay.date)}
                className={cn(
                  'flex w-full items-center justify-center gap-2 rounded-2xl border border-dashed border-neutral-200',
                  'py-3 text-sm font-medium text-neutral-400',
                  'transition-colors duration-[var(--transition-fast)] hover:border-primary-300 hover:text-primary-500'
                )}
              >
                <Plus className="h-4 w-4" aria-hidden="true" />
                Ajouter une étape
              </button>
            )}
          </>
        )}
      </div>
    </section>
  )
}

// ─── Utilitaires ──────────────────────────────────────────────────────────────

function findTodayIndex(days: ItineraryDay[]): number | null {
  const today = new Date()
  const index = days.findIndex((d) => isSameDay(new Date(d.date), today))
  return index >= 0 ? index : null
}

function isSameDay(a: Date, b: Date): boolean {
  return (
    a.getFullYear() === b.getFullYear() &&
    a.getMonth() === b.getMonth() &&
    a.getDate() === b.getDate()
  )
}

function formatDayHeading(dateStr: string, index: number): string {
  const date = new Date(dateStr)
  const isToday = isSameDay(date, new Date())
  const label = date.toLocaleDateString('fr-FR', { weekday: 'long', day: 'numeric', month: 'long' })
  return isToday ? `Aujourd'hui — ${label}` : label
}

function getStepDotColor(type: TripStep['type']): string {
  const map: Record<TripStep['type'], string> = {
    flight: 'bg-sky-400',
    accommodation: 'bg-violet-400',
    activity: 'bg-amber-400',
    transport: 'bg-teal-400',
    other: 'bg-neutral-300',
  }
  return map[type] ?? 'bg-neutral-300'
}
```

---

### 4.3 `ToolsDrawer.tsx` — Hub outils contextuel

Tiroir persistant donnant accès aux 3 outils utilitaires. S'ouvre depuis la barre de navigation basse. Pré-remplit automatiquement les champs depuis le voyage actif.

```tsx
// src/components/tools/ToolsDrawer.tsx
'use client'

import { useRef, useEffect, useState, useId } from 'react'
import { X, Languages, Coins, Clock } from 'lucide-react'
import { cn } from '@/lib/utils/cn'
import { TranslatorTool } from './TranslatorTool'
import { CurrencyConverterTool } from './CurrencyConverterTool'
import { TimezoneTool } from './TimezoneTool'
import { useActiveTrip } from '@/hooks/useActiveTrip'
import { useFocusTrap } from '@/hooks/useFocusTrap'

// ─── Types ────────────────────────────────────────────────────────────────────

type ToolTab = 'translator' | 'currency' | 'timezone'

interface ToolsDrawerProps {
  isOpen: boolean
  onClose: () => void
  defaultTab?: ToolTab
}

const TABS: { id: ToolTab; label: string; Icon: React.ElementType; shortLabel: string }[] = [
  { id: 'translator', label: 'Traducteur', shortLabel: 'Trad.', Icon: Languages },
  { id: 'currency', label: 'Convertisseur', shortLabel: 'Devises', Icon: Coins },
  { id: 'timezone', label: 'Fuseaux horaires', shortLabel: 'Heure', Icon: Clock },
]

// ─── Composant ────────────────────────────────────────────────────────────────

export function ToolsDrawer({ isOpen, onClose, defaultTab = 'translator' }: ToolsDrawerProps) {
  const [activeTab, setActiveTab] = useState<ToolTab>(defaultTab)
  const drawerRef = useRef<HTMLDivElement>(null)
  const titleId = useId()

  // Contexte voyage actif pour pré-remplissage
  const { activeTrip } = useActiveTrip()

  // Trap focus quand le drawer est ouvert (accessibilité)
  useFocusTrap(drawerRef, isOpen)

  // Fermeture au clavier Escape
  useEffect(() => {
    if (!isOpen) return
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose()
    }
    document.addEventListener('keydown', handleKeyDown)
    return () => document.removeEventListener('keydown', handleKeyDown)
  }, [isOpen, onClose])

  // Bloquer le scroll du body quand ouvert
  useEffect(() => {
    if (isOpen) {
      document.body.style.overflow = 'hidden'
    } else {
      document.body.style.overflow = ''
    }
    return () => { document.body.style.overflow = '' }
  }, [isOpen])

  // Réinitialiser l'onglet si on change de defaultTab entrant
  useEffect(() => {
    if (isOpen) setActiveTab(defaultTab)
  }, [isOpen, defaultTab])

  return (
    <>
      {/* ── Backdrop ── */}
      <div
        className={cn(
          'fixed inset-0 z-40 bg-neutral-900/40 backdrop-blur-sm',
          'transition-opacity duration-[var(--transition-base)]',
          isOpen ? 'opacity-100' : 'pointer-events-none opacity-0'
        )}
        aria-hidden="true"
        onClick={onClose}
      />

      {/* ── Drawer ── */}
      <div
        ref={drawerRef}
        role="dialog"
        aria-modal="true"
        aria-labelledby={titleId}
        className={cn(
          'fixed bottom-0 left-0 right-0 z-50 flex max-h-[85dvh] flex-col',
          'rounded-t-3xl bg-white shadow-2xl',
          'transition-transform duration-[var(--transition-base)] ease-out',
          isOpen ? 'translate-y-0' : 'translate-y-full'
        )}
      >
        {/* Poignée drag visuelle */}
        <div className="flex justify-center pt-3 pb-1" aria-hidden="true">
          <div className="h-1 w-10 rounded-full bg-neutral-200" />
        </div>

        {/* En-tête */}
        <div className="flex items-center justify-between px-5 pb-2 pt-1">
          <h2 id={titleId} className="font-display text-lg font-semibold text-neutral-900">
            Outils
          </h2>

          {/* Contexte voyage actif */}
          {activeTrip && (
            <span className="rounded-pill bg-primary-50 px-2.5 py-0.5 text-xs font-medium text-primary-700">
              {activeTrip.destination}
            </span>
          )}

          <button
            onClick={onClose}
            className={cn(
              'ml-2 flex h-8 w-8 items-center justify-center rounded-full text-neutral-400',
              'hover:bg-neutral-100 hover:text-neutral-600',
              'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500'
            )}
            aria-label="Fermer les outils"
          >
            <X className="h-4 w-4" aria-hidden="true" />
          </button>
        </div>

        {/* ── Onglets ── */}
        <div
          className="flex gap-1 border-b border-neutral-100 px-5 pb-0"
          role="tablist"
          aria-label="Sélectionner un outil"
        >
          {TABS.map(({ id, label, shortLabel, Icon }) => (
            <button
              key={id}
              role="tab"
              aria-selected={activeTab === id}
              aria-controls={`tool-panel-${id}`}
              onClick={() => setActiveTab(id)}
              className={cn(
                'flex flex-1 items-center justify-center gap-1.5 border-b-2 px-2 py-2.5 text-xs font-medium',
                'transition-colors duration-[var(--transition-fast)]',
                'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-inset focus-visible:ring-primary-500',
                activeTab === id
                  ? 'border-primary-500 text-primary-600'
                  : 'border-transparent text-neutral-500 hover:text-neutral-700'
              )}
            >
              <Icon className="h-4 w-4" aria-hidden="true" />
              <span className="hidden sm:inline">{label}</span>
              <span className="sm:hidden">{shortLabel}</span>
            </button>
          ))}
        </div>

        {/* ──

— FRONTEND ENGINEER

*(Suite et fin du livrable — reprise après la section ToolsDrawer)*

```tsx
        {/* ── Panneaux outils ── */}
        <div className="flex-1 overflow-y-auto overscroll-contain px-5 py-4">
          <div
            id="tool-panel-translator"
            role="tabpanel"
            aria-labelledby="tab-translator"
            hidden={activeTab !== 'translator'}
          >
            <TranslatorTool
              defaultTargetLanguage={activeTrip?.language ?? 'en'}
              isActive={activeTab === 'translator'}
            />
          </div>

          <div
            id="tool-panel-currency"
            role="tabpanel"
            aria-labelledby="tab-currency"
            hidden={activeTab !== 'currency'}
          >
            <CurrencyConverterTool
              defaultTargetCurrency={activeTrip?.currency ?? 'EUR'}
              isActive={activeTab === 'currency'}
            />
          </div>

          <div
            id="tool-panel-timezone"
            role="tabpanel"
            aria-labelledby="tab-timezone"
            hidden={activeTab !== 'timezone'}
          >
            <TimezoneTool
              destinationTimezone={activeTrip?.timezone ?? null}
              destinationLabel={activeTrip?.destination ?? null}
              isActive={activeTab === 'timezone'}
            />
          </div>
        </div>
      </div>
    </>
  )
}
```

---

### 4.4 `MemoryCard.tsx` — Carte souvenir du module Souvenirs

Composant d'affichage d'un souvenir (photo + note + lieu) dans la galerie. Supporte le mode lecture et le mode sélection pour le partage groupé.

```tsx
// src/components/memories/MemoryCard.tsx
'use client'

import Image from 'next/image'
import { MapPin, CheckCircle2 } from 'lucide-react'
import { cn } from '@/lib/utils/cn'
import type { Memory } from '@/types/memory'

// ─── Types ────────────────────────────────────────────────────────────────────

interface MemoryCardProps {
  memory: Memory
  isSelectable?: boolean
  isSelected?: boolean
  onSelect?: (id: string) => void
  onClick?: (id: string) => void
}

// ─── Composant ────────────────────────────────────────────────────────────────

export function MemoryCard({
  memory,
  isSelectable = false,
  isSelected = false,
  onSelect,
  onClick,
}: MemoryCardProps) {
  const handleClick = () => {
    if (isSelectable) {
      onSelect?.(memory.id)
    } else {
      onClick?.(memory.id)
    }
  }

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault()
      handleClick()
    }
  }

  return (
    <article
      role={isSelectable ? 'checkbox' : 'button'}
      aria-checked={isSelectable ? isSelected : undefined}
      aria-label={
        isSelectable
          ? `${isSelected ? 'Désélectionner' : 'Sélectionner'} le souvenir : ${memory.note ?? memory.location ?? 'Sans titre'}`
          : `Voir le souvenir : ${memory.note ?? memory.location ?? 'Sans titre'}`
      }
      tabIndex={0}
      onClick={handleClick}
      onKeyDown={handleKeyDown}
      className={cn(
        'group relative cursor-pointer overflow-hidden rounded-2xl',
        'bg-neutral-100',
        'transition-all duration-[var(--transition-fast)]',
        'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500 focus-visible:ring-offset-2',
        // Effet de scale au hover en mode galerie
        !isSelectable && 'hover:scale-[1.02] active:scale-[0.98]',
        // Bordure de sélection
        isSelected && 'ring-2 ring-primary-500 ring-offset-2'
      )}
    >
      {/* ── Image ── */}
      <div className="relative aspect-square w-full overflow-hidden">
        {memory.mediaUrl ? (
          <Image
            src={memory.mediaUrl}
            alt={memory.note ?? `Souvenir à ${memory.location ?? 'destination inconnue'}`}
            fill
            sizes="(max-width: 640px) 50vw, (max-width: 1024px) 33vw, 25vw"
            className={cn(
              'object-cover',
              'transition-transform duration-[var(--transition-slow)]',
              'group-hover:scale-105'
            )}
          />
        ) : (
          // Placeholder sans image
          <div className="flex h-full w-full items-center justify-center bg-neutral-200">
            <span className="text-3xl" aria-hidden="true">📝</span>
          </div>
        )}

        {/* Overlay gradient pour lisibilité du texte */}
        <div
          className="absolute inset-0 bg-gradient-to-t from-neutral-900/60 via-transparent to-transparent"
          aria-hidden="true"
        />

        {/* ── Badge de sélection ── */}
        {isSelectable && (
          <div
            className={cn(
              'absolute right-2 top-2 flex h-6 w-6 items-center justify-center rounded-full',
              'transition-all duration-[var(--transition-fast)]',
              isSelected
                ? 'bg-primary-500 text-white'
                : 'bg-white/80 text-neutral-400 backdrop-blur-sm'
            )}
            aria-hidden="true"
          >
            <CheckCircle2 className="h-4 w-4" />
          </div>
        )}

        {/* ── Métadonnées en bas de l'image ── */}
        <div className="absolute bottom-0 left-0 right-0 p-2.5">
          {/* Note courte */}
          {memory.note && (
            <p className="line-clamp-2 text-xs font-medium leading-snug text-white">
              {memory.note}
            </p>
          )}

          {/* Lieu */}
          {memory.location && (
            <div className="mt-0.5 flex items-center gap-1">
              <MapPin className="h-3 w-3 flex-shrink-0 text-white/70" aria-hidden="true" />
              <span className="truncate text-[10px] text-white/70">{memory.location}</span>
            </div>
          )}
        </div>
      </div>

      {/* ── Date (en dehors de l'image) ── */}
      {memory.takenAt && (
        <div className="px-2.5 py-1.5">
          <time
            dateTime={memory.takenAt}
            className="text-[10px] text-neutral-400"
          >
            {formatMemoryDate(memory.takenAt)}
          </time>
        </div>
      )}
    </article>
  )
}

// ─── Utilitaire ───────────────────────────────────────────────────────────────

function formatMemoryDate(dateStr: string): string {
  return new Date(dateStr).toLocaleDateString('fr-FR', {
    day: 'numeric',
    month: 'short',
    year: 'numeric',
  })
}
```

---

### 4.5 `OfflineBanner.tsx` — Bandeau d'état hors connexion

Composant global injecté au niveau du layout. S'affiche en haut de l'écran dès que la connectivité est perdue. Disparaît proprement à la reconnexion.

```tsx
// src/components/feedback/OfflineBanner.tsx
'use client'

import { useEffect, useState, useRef } from 'react'
import { WifiOff, Wifi } from 'lucide-react'
import { cn } from '@/lib/utils/cn'

// ─── Composant ────────────────────────────────────────────────────────────────

export function OfflineBanner() {
  const [isOffline, setIsOffline] = useState(false)
  const [showReconnected, setShowReconnected] = useState(false)
  const reconnectedTimerRef = useRef<ReturnType<typeof setTimeout> | null>(null)

  useEffect(() => {
    // Initialisation depuis l'état actuel du navigateur
    setIsOffline(!navigator.onLine)

    const handleOffline = () => {
      setIsOffline(true)
      setShowReconnected(false)
      if (reconnectedTimerRef.current) clearTimeout(reconnectedTimerRef.current)
    }

    const handleOnline = () => {
      setIsOffline(false)
      // Afficher brièvement le message de reconnexion
      setShowReconnected(true)
      reconnectedTimerRef.current = setTimeout(() => {
        setShowReconnected(false)
      }, 3000)
    }

    window.addEventListener('offline', handleOffline)
    window.addEventListener('online', handleOnline)

    return () => {
      window.removeEventListener('offline', handleOffline)
      window.removeEventListener('online', handleOnline)
      if (reconnectedTimerRef.current) clearTimeout(reconnectedTimerRef.current)
    }
  }, [])

  // Ni offline ni reconnexion récente : ne rien afficher
  if (!isOffline && !showReconnected) return null

  return (
    <div
      role="status"
      aria-live="polite"
      aria-atomic="true"
      className={cn(
        'fixed left-0 right-0 top-0 z-[100]',
        'flex items-center justify-center gap-2',
        'px-4 py-2.5 text-sm font-medium',
        'transition-transform duration-[var(--transition-base)]',
        // État offline
        isOffline && 'bg-amber-500 text-white',
        // État reconnecté
        !isOffline && showReconnected && 'bg-emerald-500 text-white'
      )}
    >
      {isOffline ? (
        <>
          <WifiOff className="h-4 w-4 flex-shrink-0" aria-hidden="true" />
          <span>Mode hors ligne — données mises en cache disponibles</span>
        </>
      ) : (
        <>
          <Wifi className="h-4 w-4 flex-shrink-0" aria-hidden="true" />
          <span>Connexion rétablie</span>
        </>
      )}
    </div>
  )
}
```

---

## 5. ARCHITECTURE FRONTEND COMPLÈTE

### 5.1 Structure des dossiers

```
src/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # Groupe de routes non authentifiées
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── register/
│   │   │   └── page.tsx
│   │   └── layout.tsx            # Layout auth (centré, fond illustré)
│   │
│   ├── (app)/                    # Groupe de routes authentifiées
│   │   ├── layout.tsx            # Layout principal (nav bottom, OfflineBanner)
│   │   ├── page.tsx              # /  → Dashboard (liste voyages)
│   │   │
│   │   ├── trips/
│   │   │   ├── new/
│   │   │   │   └── page.tsx      # Création d'un voyage
│   │   │   └── [tripId]/
│   │   │       ├── layout.tsx    # Layout voyage (header + tabs)
│   │   │       ├── page.tsx      # /trips/:id → Itinéraire (vue défaut)
│   │   │       ├── documents/
│   │   │       │   └── page.tsx
│   │   │       ├── memories/
│   │   │       │   └── page.tsx
│   │   │       └── participants/
│   │   │           └── page.tsx
│   │   │
│   │   ├── tools/
│   │   │   └── page.tsx          # Hub outils (fallback si pas de drawer)
│   │   │
│   │   ├── notifications/
│   │   │   └── page.tsx
│   │   │
│   │   └── settings/
│   │       └── page.tsx
│   │
│   ├── share/
│   │   └── [albumId]/
│   │       └── page.tsx          # Page publique partage souvenirs (non auth)
│   │
│   ├── onboarding/
│   │   └── page.tsx              # Slides value proposition
│   │
│   ├── layout.tsx                # Root layout (providers, fonts, meta)
│   └── globals.css               # Tokens CSS + Tailwind base
│
├── components/
│   ├── ui/                       # Atomes et molécules design system
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Card.tsx
│   │   ├── Badge.tsx
│   │   ├── Skeleton.tsx
│   │   ├── EmptyState.tsx
│   │   └── Modal.tsx
│   │
│   ├── layout/
│   │   ├── BottomNav.tsx         # Navigation persistante (4 onglets)
│   │   ├── TripHeader.tsx        # Header contextuel dans un voyage
│   │   └── PageTransition.tsx    # Wrapper animation entre pages
│   │
│   ├── feedback/
│   │   ├── OfflineBanner.tsx     # ✅ Codé ci-dessus
│   │   ├── Toast.tsx
│   │   └── LoadingOverlay.tsx
│   │
│   ├── trips/
│   │   ├── TripCard.tsx          # Carte voyage dans le dashboard
│   │   ├── TripForm.tsx          # Formulaire création/édition voyage
│   │   ├── TripStatusBadge.tsx   # Badge "À venir / En cours / Passé"
│   │   └── ParticipantsList.tsx
│   │
│   ├── itinerary/
│   │   ├── ItineraryView.tsx     # ✅ Codé ci-dessus
│   │   ├── DaySection.tsx        # Section d'un jour (extrait du même fichier)
│   │   ├── StepCard.tsx          # Carte d'une étape individuelle
│   │   └── StepForm.tsx          # Formulaire ajout/édition étape
│   │
│   ├── documents/
│   │   ├── DocumentList.tsx      # Liste par catégorie
│   │   ├── DocumentCard.tsx      # Carte document (icône type + nom)
│   │   ├── DocumentUploader.tsx  # Zone drag & drop + picker
│   │   └── DocumentViewer.tsx    # Visionneuse inline (PDF, image)
│   │
│   ├── tools/
│   │   ├── ToolsDrawer.tsx       # ✅ Codé ci-dessus
│   │   ├── TranslatorTool.tsx
│   │   ├── CurrencyConverterTool.tsx
│   │   └── TimezoneTool.tsx

— FRONTEND ENGINEER

*(Suite et fin du livrable — reprise après `TimezoneTool.tsx`)*

```
│   │
│   ├── memories/
│   │   ├── MemoryGallery.tsx     # ✅ Codé ci-dessus
│   │   ├── MemoryCard.tsx        # ✅ Codé ci-dessus
│   │   ├── MemoryUploader.tsx    # Ajout photo + note + lieu
│   │   └── ShareAlbumModal.tsx   # Génération + copie du lien de partage
│   │
│   └── auth/
│       ├── LoginForm.tsx
│       ├── RegisterForm.tsx
│       └── OAuthButtons.tsx      # Boutons Google + Apple
│
├── hooks/
│   ├── useActiveTrip.ts          # Trip actif depuis le store global
│   ├── useOffline.ts             # État connectivité (wraps navigator.onLine)
│   ├── useTrips.ts               # Fetching liste des voyages
│   ├── useTripDetails.ts         # Fetching détail voyage + itinéraire
│   ├── useDocuments.ts           # Fetching + upload documents
│   ├── useMemories.ts            # Fetching + upload souvenirs
│   ├── useTranslator.ts          # Appel API traduction
│   ├── useCurrencyConverter.ts   # Appel API devises + cache local
│   └── useTimezone.ts            # Calcul fuseaux depuis destination voyage
│
├── lib/
│   ├── api/
│   │   ├── client.ts             # Instance Axios centralisée (intercepteurs JWT)
│   │   ├── trips.ts              # Fonctions fetch voyages
│   │   ├── documents.ts          # Fonctions upload/fetch documents
│   │   ├── memories.ts           # Fonctions upload/fetch souvenirs
│   │   ├── tools.ts              # Fonctions traduction, devises, timezone
│   │   └── auth.ts               # Fonctions login, register, refresh
│   │
│   ├── cache/
│   │   ├── offlineCache.ts       # Lecture/écriture localStorage pour offline
│   │   └── cacheKeys.ts          # Constantes clés de cache
│   │
│   ├── store/
│   │   ├── index.ts              # Configuration Zustand
│   │   ├── authStore.ts          # État utilisateur connecté
│   │   ├── tripStore.ts          # Voyage actif + liste voyages
│   │   └── uiStore.ts            # État UI global (drawer ouvert, toast…)
│   │
│   └── utils/
│       ├── cn.ts                 # Merge Tailwind classnames (clsx + twMerge)
│       ├── formatDate.ts         # Formatage dates localisé
│       ├── formatCurrency.ts     # Formatage montants avec devise
│       └── tripStatus.ts         # Calcul statut voyage (à venir/en cours/passé)
│
├── types/
│   ├── trip.ts                   # Trip, Step, Participant
│   ├── document.ts               # TravelDocument, DocumentCategory
│   ├── memory.ts                 # Memory, Album
│   ├── tools.ts                  # TranslationResult, ExchangeRate
│   └── user.ts                   # User, AuthTokens
│
└── middleware.ts                 # Next.js middleware — protection routes auth
```

---

### 5.2 Tokens CSS — `globals.css`

```css
/* src/app/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    /* ── Palette principale ── */
    --color-primary-50:  #eff6ff;
    --color-primary-100: #dbeafe;
    --color-primary-300: #93c5fd;
    --color-primary-500: #3b82f6;
    --color-primary-600: #2563eb;
    --color-primary-700: #1d4ed8;

    /* ── Palette neutre ── */
    --color-neutral-0:   #ffffff;
    --color-neutral-50:  #f8fafc;
    --color-neutral-100: #f1f5f9;
    --color-neutral-200: #e2e8f0;
    --color-neutral-400: #94a3b8;
    --color-neutral-600: #475569;
    --color-neutral-700: #334155;
    --color-neutral-800: #1e293b;
    --color-neutral-900: #0f172a;

    /* ── Sémantique ── */
    --color-success: #10b981;
    --color-warning: #f59e0b;
    --color-error:   #ef4444;
    --color-info:    #06b6d4;

    /* ── Typographie ── */
    --font-sans: 'Inter', system-ui, sans-serif;
    --font-size-xs:   0.75rem;   /* 12px */
    --font-size-sm:   0.875rem;  /* 14px */
    --font-size-base: 1rem;      /* 16px */
    --font-size-lg:   1.125rem;  /* 18px */
    --font-size-xl:   1.25rem;   /* 20px */
    --font-size-2xl:  1.5rem;    /* 24px */
    --font-size-3xl:  1.875rem;  /* 30px */

    /* ── Espacement ── */
    --space-1:  0.25rem;
    --space-2:  0.5rem;
    --space-3:  0.75rem;
    --space-4:  1rem;
    --space-5:  1.25rem;
    --space-6:  1.5rem;
    --space-8:  2rem;
    --space-10: 2.5rem;
    --space-12: 3rem;
    --space-16: 4rem;

    /* ── Rayons ── */
    --radius-sm:   0.25rem;
    --radius-base: 0.5rem;
    --radius-md:   0.75rem;
    --radius-lg:   1rem;
    --radius-xl:   1.5rem;
    --radius-full: 9999px;

    /* ── Transitions ── */
    --transition-fast:   150ms ease;
    --transition-base:   250ms ease;
    --transition-slow:   400ms ease;

    /* ── Ombres ── */
    --shadow-sm:  0 1px 2px 0 rgb(0 0 0 / 0.05);
    --shadow-md:  0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
    --shadow-lg:  0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
    --shadow-xl:  0 20px 25px -5px rgb(0 0 0 / 0.1), 0 8px 10px -6px rgb(0 0 0 / 0.1);

    /* ── Z-index ── */
    --z-base:    0;
    --z-raised:  10;
    --z-overlay: 50;
    --z-modal:   100;
    --z-banner:  200;
  }

  /* Fond légèrement off-white pour l'ensemble de l'app */
  body {
    @apply bg-neutral-50 text-neutral-800 antialiased;
    font-family: var(--font-sans);
  }

  /* Scroll natif fluide */
  html {
    scroll-behavior: smooth;
  }

  /* Focus visible accessible */
  :focus-visible {
    @apply outline-2 outline-offset-2 outline-primary-500;
  }
}
```

---

### 5.3 Configuration Tailwind — `tailwind.config.ts`

```ts
// tailwind.config.ts
import type { Config } from 'tailwindcss'

const config: Config = {
  content: [
    './src/app/**/*.{ts,tsx}',
    './src/components/**/*.{ts,tsx}',
    './src/hooks/**/*.{ts,tsx}',
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50:  'var(--color-primary-50)',
          100: 'var(--color-primary-100)',
          300: 'var(--color-primary-300)',
          500: 'var(--color-primary-500)',
          600: 'var(--color-primary-600)',
          700: 'var(--color-primary-700)',
        },
        neutral: {
          0:   'var(--color-neutral-0)',
          50:  'var(--color-neutral-50)',
          100: 'var(--color-neutral-100)',
          200: 'var(--color-neutral-200)',
          400: 'var(--color-neutral-400)',
          600: 'var(--color-neutral-600)',
          700: 'var(--color-neutral-700)',
          800: 'var(--color-neutral-800)',
          900: 'var(--color-neutral-900)',
        },
        success: 'var(--color-success)',
        warning: 'var(--color-warning)',
        error:   'var(--color-error)',
        info:    'var(--color-info)',
      },
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
      },
      borderRadius: {
        sm:   'var(--radius-sm)',
        base: 'var(--radius-base)',
        md:   'var(--radius-md)',
        lg:   'var(--radius-lg)',
        xl:   'var(--radius-xl)',
      },
      boxShadow: {
        sm: 'var(--shadow-sm)',
        md: 'var(--shadow-md)',
        lg: 'var(--shadow-lg)',
        xl: 'var(--shadow-xl)',
      },
      // Safe area pour mobile (notch, barre système)
      spacing: {
        'safe-top':    'env(safe-area-inset-top)',
        'safe-bottom': 'env(safe-area-inset-bottom)',
        'safe-left':   'env(safe-area-inset-left)',
        'safe-right':  'env(safe-area-inset-right)',
      },
    },
  },
  plugins: [],
}

export default config
```

---

## 6. PATTERNS DE GESTION D'ÉTAT ET D'APPELS API

### 6.1 Store Zustand — `tripStore.ts`

```ts
// src/lib/store/tripStore.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import type { Trip, Step } from '@/types/trip'

// ─── Types ────────────────────────────────────────────────────────────────────

interface TripStore {
  // État
  trips: Trip[]
  activeTrip: Trip | null
  activeTripSteps: Step[]
  isLoading: boolean
  error: string | null

  // Actions
  setTrips: (trips: Trip[]) => void
  setActiveTrip: (trip: Trip | null) => void
  setActiveTripSteps: (steps: Step[]) => void
  addTrip: (trip: Trip) => void
  updateTrip: (tripId: string, updates: Partial<Trip>) => void
  setLoading: (loading: boolean) => void
  setError: (error: string | null) => void
  reset: () => void
}

// ─── Store ────────────────────────────────────────────────────────────────────

const initialState = {
  trips: [],
  activeTrip: null,
  activeTripSteps: [],
  isLoading: false,
  error: null,
}

export const useTripStore = create<TripStore>()(
  persist(
    (set) => ({
      ...initialState,

      setTrips: (trips) => set({ trips }),

      setActiveTrip: (trip) => set({ activeTrip: trip }),

      setActiveTripSteps: (steps) => set({ activeTripSteps: steps }),

      addTrip: (trip) =>
        set((state) => ({ trips: [trip, ...state.trips] })),

      updateTrip: (tripId, updates) =>
        set((state) => ({
          trips: state.trips.map((t) =>
            t.id === tripId ? { ...t, ...updates } : t
          ),
          // Si le voyage actif est mis à jour, synchroniser
          activeTrip:
            state.activeTrip?.id === tripId
              ? { ...state.activeTrip, ...updates }
              : state.activeTrip,
        })),

      setLoading: (isLoading) => set({ isLoading }),

      setError: (error) => set({ error }),

      reset: () => set(initialState),
    }),
    {
      name: 'trip-store',
      // Ne persister que les données essentielles pour l'offline
      partialize: (state) => ({
        trips: state.trips,
        activeTrip: state.activeTrip,
        activeTripSteps: state.activeTripSteps,
      }),
    }
  )
)
```

---

### 6.2 Client API — `client.ts`

```ts
// src/lib/api/client.ts
import axios, { type AxiosInstance, type AxiosError } from 'axios'
import { useAuthStore } from '@/lib/store/authStore'

// ─── Instance principale ──────────────────────────────────────────────────────

export const apiClient: AxiosInstance = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3001/v1',
  timeout: 15_000,
  headers: {
    'Content-Type': 'application/json',
    Accept: 'application/json',
  },
})

// ─── Intercepteur de requête — injection du token ─────────────────────────────

apiClient.interceptors.request.use((config) => {
  const token = useAuthStore.getState().accessToken
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// ─── Intercepteur de réponse — gestion du refresh token ──────────────────────

let isRefreshing = false
let refreshQueue: Array<(token: string) => void> = []

apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as typeof error.config & {
      _retry?: boolean
    }

    // 401 : token expiré → tentative de refresh silencieux
    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Mettre en queue les requêtes pendant le refresh
        return new Promise<string>((resolve) => {
          refreshQueue.push(resolve)
        }).then((newToken) => {
          originalRequest.headers!.Authorization = `Bearer ${newToken}`
          return apiClient(originalRequest)
        })
      }

      originalRequest._retry = true
      isRefreshing = true

      try {
        const refreshToken = useAuthStore.getState().refreshToken

— FRONTEND ENGINEER

*(Continuation — reprise après l'intercepteur de refresh token)*

```ts
        if (!refreshToken) throw new Error('No refresh token')

        const { data } = await axios.post(
          `${process.env.NEXT_PUBLIC_API_URL}/v1/auth/refresh`,
          { refreshToken }
        )

        const newAccessToken: string = data.accessToken
        useAuthStore.getState().setTokens(newAccessToken, data.refreshToken)

        // Débloquer la queue
        refreshQueue.forEach((resolve) => resolve(newAccessToken))
        refreshQueue = []

        originalRequest.headers!.Authorization = `Bearer ${newAccessToken}`
        return apiClient(originalRequest)
      } catch {
        // Refresh échoué → déconnexion
        useAuthStore.getState().logout()
        refreshQueue = []
        return Promise.reject(error)
      } finally {
        isRefreshing = false
      }
    }

    return Promise.reject(error)
  }
)
```

---

### 6.3 Hooks d'appels API — React Query

> **Pattern retenu** : TanStack Query (React Query v5) pour toute la couche data-fetching. Zustand est réservé à l'état UI global (voyage actif, préférences) et au cache offline. Ce découplage évite la duplication d'état.

```ts
// src/hooks/useTrips.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { apiClient } from '@/lib/api/client'
import { useTripStore } from '@/lib/store/tripStore'
import type { Trip, CreateTripPayload } from '@/types/trip'

// ─── Clés de cache ────────────────────────────────────────────────────────────

export const tripKeys = {
  all:    () => ['trips'] as const,
  detail: (id: string) => ['trips', id] as const,
  steps:  (id: string) => ['trips', id, 'steps'] as const,
}

// ─── Récupération de la liste des voyages ─────────────────────────────────────

export function useTrips() {
  const setTrips = useTripStore((s) => s.setTrips)

  return useQuery({
    queryKey: tripKeys.all(),
    queryFn: async () => {
      const { data } = await apiClient.get<Trip[]>('/trips')
      // Synchronisation avec le store Zustand pour l'accès offline
      setTrips(data)
      return data
    },
    staleTime: 5 * 60 * 1000,        // 5 min avant re-fetch
    gcTime:    30 * 60 * 1000,       // 30 min en cache mémoire
    retry: (failureCount, error: any) =>
      error?.response?.status !== 401 && failureCount < 2,
  })
}

// ─── Récupération d'un voyage unique ─────────────────────────────────────────

export function useTrip(tripId: string) {
  const setActiveTrip = useTripStore((s) => s.setActiveTrip)

  return useQuery({
    queryKey: tripKeys.detail(tripId),
    queryFn: async () => {
      const { data } = await apiClient.get<Trip>(`/trips/${tripId}`)
      setActiveTrip(data)
      return data
    },
    enabled: !!tripId,
    staleTime: 2 * 60 * 1000,
  })
}

// ─── Création d'un voyage ─────────────────────────────────────────────────────

export function useCreateTrip() {
  const queryClient = useQueryClient()
  const addTrip     = useTripStore((s) => s.addTrip)

  return useMutation({
    mutationFn: async (payload: CreateTripPayload) => {
      const { data } = await apiClient.post<Trip>('/trips', payload)
      return data
    },
    onSuccess: (newTrip) => {
      // Mise à jour optimiste du cache React Query
      queryClient.setQueryData<Trip[]>(tripKeys.all(), (prev = []) => [
        newTrip,
        ...prev,
      ])
      // Synchronisation Zustand
      addTrip(newTrip)
    },
    onError: (error) => {
      console.error('[useCreateTrip] Erreur création voyage :', error)
    },
  })
}
```

---

## 7. COMPOSANTS CLÉS — CODE ANNOTÉ

> Les 5 composants suivants sont les plus complexes ou les plus transversaux. Ils servent de référence pour l'implémentation du reste.

---

### 7.1 `TripCard` — Carte de voyage (Dashboard)

> Composant le plus visible de l'app. Affiché en carte principale pour le voyage actif, en liste pour les autres. Gère 3 états : `upcoming`, `active`, `past`.

```tsx
// src/components/trips/TripCard.tsx
'use client'

import Image from 'next/image'
import { useRouter } from 'next/navigation'
import { format, differenceInDays, isWithinInterval } from 'date-fns'
import { fr } from 'date-fns/locale'
import { MapPin, Calendar, Users, Wifi, WifiOff } from 'lucide-react'
import { cn } from '@/lib/utils'
import type { Trip } from '@/types/trip'

// ─── Types ────────────────────────────────────────────────────────────────────

interface TripCardProps {
  trip: Trip
  variant?: 'hero' | 'compact'    // hero = voyage actif, compact = liste
  isOffline?: boolean
  className?: string
}

type TripStatus = 'upcoming' | 'active' | 'past'

// ─── Utilitaire statut ────────────────────────────────────────────────────────

function getTripStatus(startDate: string, endDate: string): TripStatus {
  const now   = new Date()
  const start = new Date(startDate)
  const end   = new Date(endDate)

  if (isWithinInterval(now, { start, end })) return 'active'
  if (now < start) return 'upcoming'
  return 'past'
}

const STATUS_CONFIG: Record<TripStatus, { label: string; className: string }> = {
  active:   { label: 'En cours',   className: 'bg-success/10 text-success border-success/20' },
  upcoming: { label: 'À venir',    className: 'bg-primary-50 text-primary-600 border-primary-100' },
  past:     { label: 'Terminé',    className: 'bg-neutral-100 text-neutral-600 border-neutral-200' },
}

// ─── Composant ────────────────────────────────────────────────────────────────

export function TripCard({
  trip,
  variant = 'compact',
  isOffline = false,
  className,
}: TripCardProps) {
  const router = useRouter()
  const status = getTripStatus(trip.startDate, trip.endDate)
  const config = STATUS_CONFIG[status]

  const daysUntil = differenceInDays(new Date(trip.startDate), new Date())
  const duration  = differenceInDays(new Date(trip.endDate), new Date(trip.startDate))

  const handleClick = () => router.push(`/trips/${trip.id}`)

  // ── Hero variant (voyage actif, grande carte) ──────────────────────────────
  if (variant === 'hero') {
    return (
      <article
        onClick={handleClick}
        role="button"
        tabIndex={0}
        onKeyDown={(e) => e.key === 'Enter' && handleClick()}
        aria-label={`Ouvrir le voyage ${trip.name}`}
        className={cn(
          'relative overflow-hidden rounded-xl cursor-pointer',
          'bg-neutral-900 text-neutral-0',
          'shadow-xl transition-transform duration-250 ease-out',
          'hover:scale-[1.01] active:scale-[0.99]',
          'focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-primary-500',
          'min-h-[220px]',
          className
        )}
      >
        {/* Image de fond avec overlay */}
        {trip.coverImage && (
          <Image
            src={trip.coverImage}
            alt={`Photo de couverture — ${trip.name}`}
            fill
            className="object-cover opacity-40"
            priority
            sizes="(max-width: 768px) 100vw, 600px"
          />
        )}

        {/* Gradient de lisibilité */}
        <div className="absolute inset-0 bg-gradient-to-t from-neutral-900/80 via-neutral-900/20 to-transparent" />

        {/* Contenu */}
        <div className="relative z-10 p-6 flex flex-col h-full justify-between">
          {/* En-tête : statut + indicateur offline */}
          <div className="flex items-center justify-between">
            <span
              className={cn(
                'inline-flex items-center px-2.5 py-1 rounded-full text-xs font-medium border',
                config.className
              )}
            >
              {config.label}
            </span>
            {/* Indicateur de disponibilité offline */}
            <span
              aria-label={isOffline ? 'Disponible hors connexion' : 'Nécessite une connexion'}
              title={isOffline ? 'Disponible hors connexion' : 'Nécessite une connexion'}
            >
              {isOffline
                ? <WifiOff className="w-4 h-4 text-neutral-400" />
                : <Wifi className="w-4 h-4 text-neutral-400" />
              }
            </span>
          </div>

          {/* Corps */}
          <div className="space-y-2">
            <h2 className="text-2xl font-bold text-neutral-0 line-clamp-1">
              {trip.name}
            </h2>
            <div className="flex flex-wrap gap-x-4 gap-y-1 text-sm text-neutral-200">
              {/* Destination */}
              <span className="flex items-center gap-1">
                <MapPin className="w-3.5 h-3.5 shrink-0" aria-hidden />
                {trip.destination}
              </span>
              {/* Dates */}
              <span className="flex items-center gap-1">
                <Calendar className="w-3.5 h-3.5 shrink-0" aria-hidden />
                {format(new Date(trip.startDate), 'd MMM', { locale: fr })}
                {' → '}
                {format(new Date(trip.endDate), 'd MMM yyyy', { locale: fr })}
                {' · '}{duration}j
              </span>
              {/* Participants */}
              {trip.participantsCount > 1 && (
                <span className="flex items-center gap-1">
                  <Users className="w-3.5 h-3.5 shrink-0" aria-hidden />
                  {trip.participantsCount} voyageurs
                </span>
              )}
            </div>

            {/* Compte à rebours si voyage à venir */}
            {status === 'upcoming' && daysUntil > 0 && (
              <p className="text-primary-300 text-sm font-medium">
                Départ dans {daysUntil} jour{daysUntil > 1 ? 's' : ''}
              </p>
            )}
          </div>
        </div>
      </article>
    )
  }

  // ── Compact variant (liste des voyages) ────────────────────────────────────
  return (
    <article
      onClick={handleClick}
      role="button"
      tabIndex={0}
      onKeyDown={(e) => e.key === 'Enter' && handleClick()}
      aria-label={`Ouvrir le voyage ${trip.name}`}
      className={cn(
        'flex items-center gap-4 p-4 rounded-lg bg-neutral-0 border border-neutral-100',
        'shadow-sm cursor-pointer',
        'transition-all duration-150 ease-out',
        'hover:border-primary-200 hover:shadow-md',
        'active:scale-[0.98]',
        'focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-primary-500',
        className
      )}
    >
      {/* Miniature destination */}
      <div className="w-14 h-14 rounded-md overflow-hidden bg-neutral-100 shrink-0">
        {trip.coverImage ? (
          <Image
            src={trip.coverImage}
            alt=""
            width={56}
            height={56}
            className="object-cover w-full h-full"
          />
        ) : (
          <div className="w-full h-full flex items-center justify-center text-2xl">
            ✈️
          </div>
        )}
      </div>

      {/* Infos */}
      <div className="flex-1 min-w-0">
        <h3 className="font-semibold text-neutral-800 truncate">{trip.name}</h3>
        <p className="text-sm text-neutral-600 flex items-center gap-1 mt-0.5">
          <MapPin className="w-3 h-3 shrink-0" aria-hidden />
          <span className="truncate">{trip.destination}</span>
        </p>
        <p className="text-xs text-neutral-400 mt-0.5">
          {format(new Date(trip.startDate), 'd MMM', { locale: fr })}
          {' — '}
          {format(new Date(trip.endDate), 'd MMM yyyy', { locale: fr })}
        </p>
      </div>

      {/* Badge statut */}
      <span
        className={cn(
          'shrink-0 inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium border',
          config.className
        )}
        aria-label={`Statut : ${config.label}`}
      >
        {config.label}
      </span>
    </article>
  )
}
```

---

### 7.2 `ItineraryDayView` — Vue jour de l'itinéraire

> Composant central de l'app. Affiche les étapes d'une journée en timeline verticale. Gère les états vides, les étapes passées/à venir, et l'accès aux documents associés.

```tsx
// src/components/itinerary/ItineraryDayView.tsx
'use client'

import { useState } from 'react'
import { format } from 'date-fns'
import { fr } from 'date-fns/locale'
import {
  Plane, Hotel, Utensils, Camera, MapPin,
  Clock, FileText, ChevronRight, Plus
} from 'lucide-react'
import { cn } from '@/lib/utils'
import { EmptyState } from '@/components/ui/EmptyState'
import type { Step, StepCategory } from '@/types/trip'

// ─── Types ────────────────────────────────────────────────────────────────────

interface ItineraryDayViewProps {
  date: string           // ISO date string (ex: "2024-07-15")
  steps: Step[]
  isToday: boolean
  canEdit?: boolean
  onAddStep?: () => void
  onStepClick?: (

— FRONTEND ENGINEER

*(continuation exacte depuis le dernier mot produit)*

```tsx
  onStepClick?: (stepId: string) => void
}

// ─── Icônes par catégorie ─────────────────────────────────────────────────────

const STEP_ICONS: Record<StepCategory, React.ElementType> = {
  flight:       Plane,
  accommodation: Hotel,
  restaurant:   Utensils,
  activity:     Camera,
  transport:    MapPin,
  other:        MapPin,
}

const STEP_COLORS: Record<StepCategory, string> = {
  flight:        'bg-blue-100 text-blue-600 border-blue-200',
  accommodation: 'bg-amber-100 text-amber-600 border-amber-200',
  restaurant:    'bg-rose-100 text-rose-600 border-rose-200',
  activity:      'bg-green-100 text-green-600 border-green-200',
  transport:     'bg-purple-100 text-purple-600 border-purple-200',
  other:         'bg-neutral-100 text-neutral-600 border-neutral-200',
}

// ─── Sous-composant : StepItem ────────────────────────────────────────────────

interface StepItemProps {
  step: Step
  isPast: boolean
  isNext: boolean
  onClick?: () => void
}

function StepItem({ step, isPast, isNext, onClick }: StepItemProps) {
  const Icon = STEP_ICONS[step.category]
  const colorClass = STEP_COLORS[step.category]

  return (
    <li
      className={cn(
        'relative flex gap-4 pb-6 last:pb-0',
        isPast && 'opacity-50'
      )}
    >
      {/* Ligne verticale de timeline */}
      <div
        aria-hidden
        className="absolute left-5 top-10 bottom-0 w-px bg-neutral-200 last:hidden"
      />

      {/* Icône catégorie */}
      <div
        className={cn(
          'relative z-10 w-10 h-10 rounded-full border flex items-center justify-center shrink-0 mt-1',
          colorClass
        )}
        aria-hidden
      >
        <Icon className="w-4 h-4" />
      </div>

      {/* Contenu de l'étape */}
      <button
        onClick={onClick}
        className={cn(
          'flex-1 min-w-0 text-left rounded-xl border bg-neutral-0 p-4',
          'shadow-sm transition-all duration-150',
          'hover:border-primary-200 hover:shadow-md',
          'active:scale-[0.98]',
          'focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-primary-500',
          isNext && 'border-primary-300 ring-2 ring-primary-100'
        )}
        aria-label={`Étape : ${step.title}${isNext ? ' — prochaine étape' : ''}`}
      >
        <div className="flex items-start justify-between gap-2">
          <div className="min-w-0">
            {/* Badge "Prochaine" */}
            {isNext && (
              <span className="inline-flex items-center px-2 py-0.5 rounded-full text-xs font-medium bg-primary-50 text-primary-600 border border-primary-100 mb-1.5">
                Prochaine étape
              </span>
            )}

            <h3 className="font-semibold text-neutral-800 truncate">
              {step.title}
            </h3>

            {/* Horaire */}
            {step.time && (
              <p className="flex items-center gap-1 text-sm text-neutral-500 mt-0.5">
                <Clock className="w-3.5 h-3.5 shrink-0" aria-hidden />
                {step.time}
                {step.endTime && ` → ${step.endTime}`}
              </p>
            )}

            {/* Lieu */}
            {step.location && (
              <p className="flex items-center gap-1 text-sm text-neutral-500 mt-0.5">
                <MapPin className="w-3.5 h-3.5 shrink-0" aria-hidden />
                <span className="truncate">{step.location}</span>
              </p>
            )}

            {/* Note courte */}
            {step.note && (
              <p className="text-sm text-neutral-400 mt-1 line-clamp-2">
                {step.note}
              </p>
            )}

            {/* Documents associés */}
            {step.documentsCount > 0 && (
              <p className="flex items-center gap-1 text-xs text-neutral-400 mt-2">
                <FileText className="w-3 h-3 shrink-0" aria-hidden />
                {step.documentsCount} document{step.documentsCount > 1 ? 's' : ''}
              </p>
            )}
          </div>

          <ChevronRight className="w-4 h-4 text-neutral-300 shrink-0 mt-1" aria-hidden />
        </div>
      </button>
    </li>
  )
}

// ─── Composant principal ──────────────────────────────────────────────────────

export function ItineraryDayView({
  date,
  steps,
  isToday,
  canEdit = false,
  onAddStep,
  onStepClick,
}: ItineraryDayViewProps) {
  const now = new Date()

  // Détermination de la prochaine étape (uniquement si c'est aujourd'hui)
  const nextStepId = isToday
    ? steps.find((s) => s.time && new Date(`${date}T${s.time}`) > now)?.id
    : null

  if (steps.length === 0) {
    return (
      <div className="py-8">
        <EmptyState
          icon="📅"
          title="Aucune étape ce jour"
          description={canEdit ? 'Ajoutez votre première étape pour ce jour.' : 'Aucune activité prévue.'}
          action={
            canEdit && onAddStep
              ? { label: 'Ajouter une étape', onClick: onAddStep }
              : undefined
          }
        />
      </div>
    )
  }

  return (
    <section aria-label={`Itinéraire du ${format(new Date(date), 'EEEE d MMMM', { locale: fr })}`}>
      {/* En-tête de la journée */}
      <div className="flex items-center justify-between mb-6">
        <div>
          <h2 className="text-lg font-bold text-neutral-800 capitalize">
            {format(new Date(date), 'EEEE d MMMM', { locale: fr })}
          </h2>
          {isToday && (
            <span className="text-sm text-primary-600 font-medium">Aujourd'hui</span>
          )}
        </div>
        {canEdit && onAddStep && (
          <button
            onClick={onAddStep}
            className={cn(
              'flex items-center gap-1.5 px-3 py-1.5 rounded-lg text-sm font-medium',
              'bg-primary-50 text-primary-600 border border-primary-100',
              'hover:bg-primary-100 transition-colors',
              'focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-primary-500'
            )}
            aria-label="Ajouter une étape à ce jour"
          >
            <Plus className="w-4 h-4" aria-hidden />
            Ajouter
          </button>
        )}
      </div>

      {/* Liste des étapes */}
      <ol className="relative" aria-label="Étapes de la journée">
        {steps
          .sort((a, b) => (a.time ?? '').localeCompare(b.time ?? ''))
          .map((step) => {
            const isPast =
              isToday && step.time
                ? new Date(`${date}T${step.time}`) < now
                : !isToday && new Date(date) < now
            const isNext = step.id === nextStepId

            return (
              <StepItem
                key={step.id}
                step={step}
                isPast={isPast}
                isNext={isNext}
                onClick={() => onStepClick?.(step.id)}
              />
            )
          })}
      </ol>
    </section>
  )
}
```

---

### 7.3 `ToolsDrawer` — Hub outils utilitaires

> Composant de fond de tiroir (bottom sheet) persistant donnant accès aux trois outils contextualisés. Récupère automatiquement la devise et la langue depuis le store voyage actif.

```tsx
// src/components/tools/ToolsDrawer.tsx
'use client'

import { useState, useRef, useEffect } from 'react'
import { Languages, ArrowLeftRight, Clock, ChevronDown, X } from 'lucide-react'
import { cn } from '@/lib/utils'
import { useActiveTrip } from '@/hooks/useActiveTrip'
import { TranslatorPanel } from './TranslatorPanel'
import { CurrencyPanel } from './CurrencyPanel'
import { TimezonePanel } from './TimezonePanel'

// ─── Types ────────────────────────────────────────────────────────────────────

type ToolTab = 'translator' | 'currency' | 'timezone'

const TABS: Array<{ id: ToolTab; label: string; Icon: React.ElementType; ariaLabel: string }> = [
  { id: 'translator', label: 'Traducteur',   Icon: Languages,      ariaLabel: 'Ouvrir le traducteur' },
  { id: 'currency',   label: 'Devises',       Icon: ArrowLeftRight, ariaLabel: 'Ouvrir le convertisseur de devises' },
  { id: 'timezone',   label: 'Fuseaux',       Icon: Clock,          ariaLabel: 'Ouvrir les fuseaux horaires' },
]

interface ToolsDrawerProps {
  isOpen: boolean
  onClose: () => void
}

// ─── Composant ────────────────────────────────────────────────────────────────

export function ToolsDrawer({ isOpen, onClose }: ToolsDrawerProps) {
  const [activeTab, setActiveTab] = useState<ToolTab>('translator')
  const drawerRef = useRef<HTMLDivElement>(null)
  const closeButtonRef = useRef<HTMLButtonElement>(null)

  // Contexte du voyage actif : devise + langue destination
  const { activeTrip } = useActiveTrip()

  // Focus trap : focus sur le bouton fermer à l'ouverture
  useEffect(() => {
    if (isOpen) {
      closeButtonRef.current?.focus()
    }
  }, [isOpen])

  // Fermeture au clavier (Escape)
  useEffect(() => {
    function handleKeyDown(e: KeyboardEvent) {
      if (e.key === 'Escape' && isOpen) onClose()
    }
    document.addEventListener('keydown', handleKeyDown)
    return () => document.removeEventListener('keydown', handleKeyDown)
  }, [isOpen, onClose])

  return (
    <>
      {/* Overlay */}
      <div
        aria-hidden
        onClick={onClose}
        className={cn(
          'fixed inset-0 bg-neutral-900/40 z-40 transition-opacity duration-300',
          isOpen ? 'opacity-100 pointer-events-auto' : 'opacity-0 pointer-events-none'
        )}
      />

      {/* Drawer */}
      <div
        ref={drawerRef}
        role="dialog"
        aria-modal="true"
        aria-label="Outils de voyage"
        className={cn(
          'fixed bottom-0 left-0 right-0 z-50',
          'bg-neutral-0 rounded-t-2xl shadow-2xl',
          'transition-transform duration-300 ease-out',
          'max-h-[85dvh] flex flex-col',
          isOpen ? 'translate-y-0' : 'translate-y-full'
        )}
      >
        {/* Handle + Header */}
        <div className="flex flex-col items-center pt-3 pb-2 px-6 border-b border-neutral-100">
          {/* Drag handle visuel */}
          <div className="w-10 h-1 bg-neutral-200 rounded-full mb-4" aria-hidden />

          <div className="w-full flex items-center justify-between">
            <h2 className="text-lg font-bold text-neutral-800">Outils</h2>
            <button
              ref={closeButtonRef}
              onClick={onClose}
              className={cn(
                'w-8 h-8 rounded-full flex items-center justify-center',
                'text-neutral-500 hover:bg-neutral-100',
                'focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-primary-500',
                'transition-colors'
              )}
              aria-label="Fermer les outils"
            >
              <X className="w-4 h-4" />
            </button>
          </div>

          {/* Bandeau contextuel voyage actif */}
          {activeTrip && (
            <p className="w-full text-xs text-neutral-400 mt-1">
              Contexte : <span className="font-medium text-neutral-600">{activeTrip.destination}</span>
              {activeTrip.currency && (
                <> · <span className="font-medium text-neutral-600">{activeTrip.currency}</span></>
              )}
              {activeTrip.language && (
                <> · <span className="font-medium text-neutral-600">{activeTrip.language}</span></>
              )}
            </p>
          )}
        </div>

        {/* Tabs */}
        <div
          role="tablist"
          aria-label="Sélection de l'outil"
          className="flex border-b border-neutral-100 px-6"
        >
          {TABS.map(({ id, label, Icon, ariaLabel }) => (
            <button
              key={id}
              role="tab"
              aria-selected={activeTab === id}
              aria-controls={`panel-${id}`}
              id={`tab-${id}`}
              onClick={() => setActiveTab(id)}
              className={cn(
                'flex-1 flex flex-col items-center gap-1 py-3 text-xs font-medium',
                'border-b-2 transition-colors duration-150',
                activeTab === id
                  ? 'border-primary-500 text-primary-600'
                  : 'border-transparent text-neutral-400 hover:text-neutral-600'
              )}
              aria-label={ariaLabel}
            >
              <Icon className="w-5 h-5" aria-hidden />
              {label}
            </button>
          ))}
        </div>

        {/* Panneaux */}
        <div className="flex-1 overflow-y-auto overscroll-contain">
          <div
            role="tabpanel"
            id="panel-translator"
            aria-labelledby="tab-translator"
            hidden={activeTab !== 'translator'}
            className="p-6"
          >
            <TranslatorPanel
              defaultTargetLanguage={

— FRONTEND ENGINEER

*(suite exacte du composant `ToolsDrawer`)*

```tsx
                activeTrip?.language ?? 'en'
              }
            />
          </div>

          <div
            role="tabpanel"
            id="panel-currency"
            aria-labelledby="tab-currency"
            hidden={activeTab !== 'currency'}
            className="p-6"
          >
            <CurrencyPanel
              defaultFromCurrency={activeTrip?.homeCurrency ?? 'EUR'}
              defaultToCurrency={activeTrip?.currency ?? 'USD'}
            />
          </div>

          <div
            role="tabpanel"
            id="panel-timezone"
            aria-labelledby="tab-timezone"
            hidden={activeTab !== 'timezone'}
            className="p-6"
          >
            <TimezonePanel
              destinationTimezone={activeTrip?.timezone ?? 'UTC'}
              destinationLabel={activeTrip?.destination ?? 'Destination'}
            />
          </div>
        </div>
      </div>
    </>
  )
}
```

---

### 7.4 `CurrencyPanel` — Convertisseur de devises contextualisé

> Panneau enfant de `ToolsDrawer`. Pré-remplit les devises depuis le contexte voyage. Affiche le taux et sa date de mise à jour. Gère l'état offline proprement.

```tsx
// src/components/tools/CurrencyPanel.tsx
'use client'

import { useState, useEffect, useCallback } from 'react'
import { ArrowLeftRight, RefreshCw, WifiOff } from 'lucide-react'
import { cn } from '@/lib/utils'
import { useCurrencyRates } from '@/hooks/useCurrencyRates'
import { formatCurrency } from '@/lib/formatters'

interface CurrencyPanelProps {
  defaultFromCurrency: string
  defaultToCurrency: string
}

export function CurrencyPanel({ defaultFromCurrency, defaultToCurrency }: CurrencyPanelProps) {
  const [amount, setAmount] = useState<string>('1')
  const [from, setFrom] = useState(defaultFromCurrency)
  const [to, setTo] = useState(defaultToCurrency)

  // Hook qui gère le cache Redis côté API et le cache local (stale-while-revalidate)
  const { rate, updatedAt, isLoading, isOffline, refetch } = useCurrencyRates(from, to)

  // Résultat converti
  const numericAmount = parseFloat(amount) || 0
  const converted = rate ? numericAmount * rate : null

  // Inverser les devises
  const handleSwap = useCallback(() => {
    setFrom(to)
    setTo(from)
  }, [from, to])

  // Sync si les props changent (changement de voyage actif)
  useEffect(() => {
    setFrom(defaultFromCurrency)
    setTo(defaultToCurrency)
  }, [defaultFromCurrency, defaultToCurrency])

  return (
    <div className="space-y-5">
      {/* Bandeau offline */}
      {isOffline && (
        <div
          role="status"
          className="flex items-center gap-2 px-3 py-2 rounded-lg bg-amber-50 border border-amber-100 text-amber-700 text-sm"
        >
          <WifiOff className="w-4 h-4 shrink-0" aria-hidden />
          <span>Taux affichés depuis le cache · Mise à jour impossible hors connexion</span>
        </div>
      )}

      {/* Montant source */}
      <div>
        <label htmlFor="currency-amount" className="block text-xs font-medium text-neutral-500 mb-1.5">
          Montant
        </label>
        <div className="relative">
          <input
            id="currency-amount"
            type="number"
            inputMode="decimal"
            min="0"
            value={amount}
            onChange={(e) => setAmount(e.target.value)}
            className={cn(
              'w-full px-4 py-3 rounded-xl border border-neutral-200',
              'text-2xl font-bold text-neutral-800 bg-neutral-50',
              'focus:outline-none focus:ring-2 focus:ring-primary-400 focus:border-transparent',
              'transition'
            )}
            aria-label={`Montant en ${from}`}
          />
          <span
            className="absolute right-4 top-1/2 -translate-y-1/2 text-sm font-semibold text-neutral-400"
            aria-hidden
          >
            {from}
          </span>
        </div>
      </div>

      {/* Bouton swap */}
      <div className="flex justify-center">
        <button
          onClick={handleSwap}
          className={cn(
            'flex items-center gap-2 px-4 py-2 rounded-full',
            'border border-neutral-200 bg-white text-neutral-600',
            'hover:bg-neutral-50 hover:border-neutral-300',
            'focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-primary-500',
            'transition-all text-sm font-medium'
          )}
          aria-label={`Inverser les devises : passer de ${from} vers ${to}`}
        >
          <ArrowLeftRight className="w-4 h-4" aria-hidden />
          Inverser
        </button>
      </div>

      {/* Résultat */}
      <div
        className="p-4 rounded-xl bg-primary-50 border border-primary-100"
        aria-live="polite"
        aria-atomic="true"
      >
        {isLoading ? (
          <div className="flex items-center gap-2 text-neutral-400 text-sm">
            <RefreshCw className="w-4 h-4 animate-spin" aria-hidden />
            Chargement du taux…
          </div>
        ) : converted !== null ? (
          <>
            <p className="text-3xl font-bold text-primary-700">
              {formatCurrency(converted, to)}
            </p>
            <p className="text-sm text-neutral-500 mt-1">
              1 {from} = {rate?.toFixed(4)} {to}
            </p>
          </>
        ) : (
          <p className="text-sm text-neutral-400">Taux non disponible</p>
        )}
      </div>

      {/* Méta taux */}
      <div className="flex items-center justify-between">
        <p className="text-xs text-neutral-400">
          {updatedAt
            ? `Taux mis à jour le ${updatedAt}`
            : 'Taux non chargé'}
        </p>
        {!isOffline && (
          <button
            onClick={refetch}
            className={cn(
              'text-xs text-primary-500 hover:text-primary-700',
              'flex items-center gap-1',
              'focus-visible:outline-2 focus-visible:outline-offset-2 focus-visible:outline-primary-500'
            )}
            aria-label="Rafraîchir le taux de change"
          >
            <RefreshCw className="w-3 h-3" aria-hidden />
            Rafraîchir
          </button>
        )}
      </div>
    </div>
  )
}
```

---

### 7.5 `TripCard` — Carte voyage sur le tableau de bord

> Composant carte affiché dans le tableau de bord. Trois variantes visuelles selon le statut (upcoming / active / past). Lien de navigation vers la vue voyage.

```tsx
// src/components/dashboard/TripCard.tsx
import Link from 'next/link'
import Image from 'next/image'
import { Calendar, Users, MapPin, ChevronRight } from 'lucide-react'
import { cn } from '@/lib/utils'
import { formatDateRange } from '@/lib/formatters'
import type { Trip } from '@/types/trip'

// ─── Types ────────────────────────────────────────────────────────────────────

type TripStatus = 'upcoming' | 'active' | 'past'

interface TripCardProps {
  trip: Trip
  variant?: 'featured' | 'compact'
}

// ─── Utilitaires ──────────────────────────────────────────────────────────────

function getTripStatus(startDate: string, endDate: string): TripStatus {
  const now = new Date()
  const start = new Date(startDate)
  const end = new Date(endDate)
  if (now < start) return 'upcoming'
  if (now > end) return 'past'
  return 'active'
}

const STATUS_CONFIG: Record<TripStatus, { label: string; className: string }> = {
  active:   { label: 'En cours',  className: 'bg-green-50 text-green-700 border-green-100' },
  upcoming: { label: 'À venir',   className: 'bg-primary-50 text-primary-700 border-primary-100' },
  past:     { label: 'Terminé',   className: 'bg-neutral-100 text-neutral-500 border-neutral-200' },
}

// ─── Composant ────────────────────────────────────────────────────────────────

export function TripCard({ trip, variant = 'compact' }: TripCardProps) {
  const status = getTripStatus(trip.startDate, trip.endDate)
  const { label, className: statusClass } = STATUS_CONFIG[status]
  const isFeatured = variant === 'featured'
  const isPast = status === 'past'

  return (
    <Link
      href={`/trips/${trip.id}`}
      className={cn(
        'group block rounded-2xl overflow-hidden border',
        'transition-all duration-200',
        isFeatured
          ? 'border-transparent shadow-lg hover:shadow-xl hover:-translate-y-0.5'
          : 'border-neutral-200 bg-white hover:border-neutral-300 hover:shadow-md',
        isPast && 'opacity-75 hover:opacity-100'
      )}
      aria-label={`Voyage ${trip.name} — ${label}`}
    >
      {/* Image de couverture (featured uniquement) */}
      {isFeatured && (
        <div className="relative h-44 bg-neutral-200">
          {trip.coverImageUrl ? (
            <Image
              src={trip.coverImageUrl}
              alt={`Photo de ${trip.destination}`}
              fill
              className="object-cover"
              sizes="(max-width: 768px) 100vw, 600px"
              priority
            />
          ) : (
            <div className="absolute inset-0 bg-gradient-to-br from-primary-400 to-primary-600 flex items-center justify-center">
              <MapPin className="w-12 h-12 text-white/60" aria-hidden />
            </div>
          )}
          {/* Overlay gradient pour lisibilité du badge */}
          <div className="absolute inset-0 bg-gradient-to-t from-black/40 to-transparent" aria-hidden />

          {/* Badge statut */}
          <span
            className={cn(
              'absolute top-3 right-3 px-2.5 py-1 rounded-full text-xs font-semibold border',
              statusClass
            )}
            aria-label={`Statut : ${label}`}
          >
            {label}
          </span>
        </div>
      )}

      {/* Corps de la carte */}
      <div className={cn('p-4', isFeatured && 'bg-white')}>
        <div className="flex items-start justify-between gap-3">
          <div className="min-w-0 flex-1">
            {/* Destination */}
            <div className="flex items-center gap-1.5 mb-1">
              <MapPin className="w-3.5 h-3.5 text-primary-400 shrink-0" aria-hidden />
              <span className="text-xs font-medium text-primary-600 truncate">
                {trip.destination}
              </span>
            </div>

            {/* Nom du voyage */}
            <h3
              className={cn(
                'font-bold text-neutral-800 truncate',
                isFeatured ? 'text-xl' : 'text-base'
              )}
            >
              {trip.name}
            </h3>

            {/* Dates */}
            <div className="flex items-center gap-1 mt-1.5 text-sm text-neutral-500">
              <Calendar className="w-3.5 h-3.5 shrink-0" aria-hidden />
              <time dateTime={`${trip.startDate}/${trip.endDate}`}>
                {formatDateRange(trip.startDate, trip.endDate)}
              </time>
            </div>
          </div>

          {/* Statut (compact) + chevron */}
          <div className="flex flex-col items-end gap-2 shrink-0">
            {!isFeatured && (
              <span
                className={cn('px-2 py-0.5 rounded-full text-xs font-semibold border', statusClass)}
              >
                {label}
              </span>
            )}
            <ChevronRight
              className="w-4 h-4 text-neutral-300 group-hover:text-neutral-500 transition-colors mt-auto"
              aria-hidden
            />
          </div>
        </div>

        {/* Participants */}
        {trip.participantCount > 1 && (
          <div className="flex items-center gap-1 mt-3 text-xs text-neutral-400">
            <Users className="w-3.5 h-3.5" aria-hidden />
            <span>
              {trip.participantCount} voyageur{trip.participantCount > 1 ? 's' : ''}
            </span>
          </div>
        )}
      </div>
    </Link>
  )
}
```

---

## 8. PATTERNS DE GESTION D'ÉTAT ET D'APPELS API

### 8.1 Architecture des stores (Zustand)

```
src/store/
├── authStore.ts          // session, user, tokens
├── tripStore.ts          // voyage actif, liste des voyages
├── uiStore.ts            // drawer ouvert, onglet actif, toasts
└── offlineStore.ts       // statut réseau, queue de sync
```

#### `tripStore.ts` — Store central voyage

```ts
// src/store/tripStore.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import type { Trip, Step } from '@/types/trip'

interface TripState {
  // État
  trips: Trip[]
  activeTripId: string | null
  stepsCache: Record<string, Step[]>   // clé : tripId

  // Sélecteurs calculés
  activeTrip: Trip | null

  // Actions
  setTrips: (trips: Trip[]) => void
  setActiveTripId: (id: string | null) => void
  upsertTrip: (trip: Trip) => void
  setSteps: (tripId: string, steps: Step[]) => void
  upsertStep: (tripId: string, step: Step) => void
  removeStep: (tripId: string, stepId: string) => void
}

export const useTripStore = create<TripState>()(
  persist(
    (set, get) => ({
      trips: [],
      activeTripId: null,
      stepsCache: {},

      //

— FRONTEND ENGINEER

*(Suite et fin du livrable)*

```ts
      // Sélecteur calculé
      get activeTrip() {
        const { trips, activeTripId } = get()
        return trips.find(t => t.id === activeTripId) ?? null
      },

      // Actions
      setTrips: (trips) => set({ trips }),
      setActiveTripId: (id) => set({ activeTripId: id }),

      upsertTrip: (trip) =>
        set((state) => {
          const exists = state.trips.findIndex(t => t.id === trip.id)
          if (exists >= 0) {
            const updated = [...state.trips]
            updated[exists] = trip
            return { trips: updated }
          }
          return { trips: [...state.trips, trip] }
        }),

      setSteps: (tripId, steps) =>
        set((state) => ({
          stepsCache: { ...state.stepsCache, [tripId]: steps },
        })),

      upsertStep: (tripId, step) =>
        set((state) => {
          const current = state.stepsCache[tripId] ?? []
          const idx = current.findIndex(s => s.id === step.id)
          const updated = idx >= 0
            ? current.map(s => s.id === step.id ? step : s)
            : [...current, step].sort((a, b) =>
                new Date(a.startTime).getTime() - new Date(b.startTime).getTime()
              )
          return {
            stepsCache: { ...state.stepsCache, [tripId]: updated },
          }
        }),

      removeStep: (tripId, stepId) =>
        set((state) => ({
          stepsCache: {
            ...state.stepsCache,
            [tripId]: (state.stepsCache[tripId] ?? []).filter(s => s.id !== stepId),
          },
        })),
    }),
    {
      name: 'trip-store',
      // On persiste uniquement les données légères ; les documents/médias
      // restent dans le cache HTTP et le stockage local dédié.
      partialize: (state) => ({
        trips: state.trips,
        activeTripId: state.activeTripId,
        stepsCache: state.stepsCache,
      }),
    }
  )
)
```

---

### 8.2 Pattern d'appels API — React Query

Tous les appels serveur passent par **TanStack Query (React Query v5)**. Le client Axios est configuré une fois avec les intercepteurs JWT.

```ts
// src/lib/apiClient.ts
import axios from 'axios'
import { useAuthStore } from '@/store/authStore'

export const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  timeout: 10_000,
})

// Injecte le token sur chaque requête
apiClient.interceptors.request.use((config) => {
  const token = useAuthStore.getState().accessToken
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

// Gère l'expiration du token silencieusement
apiClient.interceptors.response.use(
  (res) => res,
  async (error) => {
    const original = error.config
    if (error.response?.status === 401 && !original._retry) {
      original._retry = true
      try {
        await useAuthStore.getState().refreshTokens()
        return apiClient(original)
      } catch {
        useAuthStore.getState().logout()
      }
    }
    return Promise.reject(error)
  }
)
```

#### Hook type — `useTrip`

```ts
// src/hooks/useTrip.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { apiClient } from '@/lib/apiClient'
import { useTripStore } from '@/store/tripStore'
import type { Trip } from '@/types/trip'

// Clés de cache centralisées
export const tripKeys = {
  all:    () => ['trips'] as const,
  detail: (id: string) => ['trips', id] as const,
  steps:  (id: string) => ['trips', id, 'steps'] as const,
}

export function useTrips() {
  const setTrips = useTripStore(s => s.setTrips)

  return useQuery({
    queryKey: tripKeys.all(),
    queryFn: async () => {
      const { data } = await apiClient.get<Trip[]>('/v1/trips')
      setTrips(data)                // synchronise le store Zustand
      return data
    },
    staleTime: 5 * 60 * 1000,       // 5 min avant refetch
    gcTime:    30 * 60 * 1000,      // 30 min en cache
  })
}

export function useTrip(tripId: string) {
  const upsertTrip = useTripStore(s => s.upsertTrip)

  return useQuery({
    queryKey: tripKeys.detail(tripId),
    queryFn: async () => {
      const { data } = await apiClient.get<Trip>(`/v1/trips/${tripId}`)
      upsertTrip(data)
      return data
    },
    enabled: Boolean(tripId),
  })
}

export function useCreateTrip() {
  const qc = useQueryClient()
  const upsertTrip = useTripStore(s => s.upsertTrip)
  const setActiveTripId = useTripStore(s => s.setActiveTripId)

  return useMutation({
    mutationFn: async (payload: Omit<Trip, 'id' | 'createdAt'>) => {
      const { data } = await apiClient.post<Trip>('/v1/trips', payload)
      return data
    },
    onSuccess: (trip) => {
      upsertTrip(trip)
      setActiveTripId(trip.id)
      // Invalide la liste pour forcer un refetch
      qc.invalidateQueries({ queryKey: tripKeys.all() })
    },
  })
}
```

#### Stratégie offline — `networkMode`

```ts
// src/lib/queryClientConfig.ts
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // Utilise le cache même hors connexion, tente le réseau si dispo
      networkMode: 'offlineFirst',
      retry: (failureCount, error: any) => {
        // Pas de retry sur 401/403/404
        if ([401, 403, 404].includes(error?.response?.status)) return false
        return failureCount < 2
      },
    },
    mutations: {
      // Les mutations s'exécutent uniquement si connecté
      networkMode: 'online',
    },
  },
})
```

---

### 8.3 Gestion du statut offline

```ts
// src/hooks/useNetworkStatus.ts
'use client'
import { useEffect, useState } from 'react'
import { useOfflineStore } from '@/store/offlineStore'

export function useNetworkStatus() {
  const [isOnline, setIsOnline] = useState(
    typeof navigator !== 'undefined' ? navigator.onLine : true
  )
  const setOffline = useOfflineStore(s => s.setIsOffline)

  useEffect(() => {
    const handleOnline  = () => { setIsOnline(true);  setOffline(false) }
    const handleOffline = () => { setIsOnline(false); setOffline(true)  }

    window.addEventListener('online',  handleOnline)
    window.addEventListener('offline', handleOffline)
    return () => {
      window.removeEventListener('online',  handleOnline)
      window.removeEventListener('offline', handleOffline)
    }
  }, [setOffline])

  return { isOnline }
}
```

```tsx
// src/components/ui/OfflineBanner.tsx
'use client'
import { WifiOff } from 'lucide-react'
import { useOfflineStore } from '@/store/offlineStore'
import { cn } from '@/lib/utils'

export function OfflineBanner() {
  const isOffline = useOfflineStore(s => s.isOffline)

  return (
    <div
      role="status"
      aria-live="polite"
      aria-label={isOffline ? 'Mode hors connexion actif' : undefined}
      className={cn(
        'fixed top-0 inset-x-0 z-50 flex items-center justify-center gap-2',
        'px-4 py-2 text-sm font-medium text-white bg-amber-500',
        'transition-transform duration-300',
        isOffline ? 'translate-y-0' : '-translate-y-full'
      )}
    >
      <WifiOff className="w-4 h-4 shrink-0" aria-hidden />
      Mode hors connexion — Données locales affichées
    </div>
  )
}
```

---

## 9. POINTS D'ATTENTION TECHNIQUES

### 9.1 Performance

| Risque | Mesure |
|---|---|
| Chargement initial lent sur mobile | Route-based code splitting Next.js (`dynamic()` sur tous les modules outils) |
| Images de voyage volumineuses | `next/image` avec `sizes` adaptatifs + format WebP automatique |
| Liste d'étapes longue (itinéraire) | Virtualisation avec `@tanstack/react-virtual` si > 30 items |
| Re-renders fréquents sur le convertisseur | `useDeferredValue` sur le champ montant, debounce 300 ms |
| Taux de change re-fetché trop souvent | `staleTime: 60 * 60 * 1000` (1h) + cache Redis côté serveur |
| Fonts bloquantes | `next/font` avec `display: swap`, subset latin + caractères destination |

### 9.2 Accessibilité (WCAG 2.1 AA)

- **Navigation clavier** : tous les éléments interactifs accessibles au Tab/Enter/Space, ordre logique dans le DOM
- **Focus visible** : ring Tailwind `focus-visible:outline-2 focus-visible:outline-primary-500` systématique — jamais supprimé sans alternative
- **Contrastes** : tokens couleurs vérifiés (ratio ≥ 4.5:1 texte normal, ≥ 3:1 grands textes et UI) — le `text-neutral-400` est à surveiller sur fond blanc
- **Labels formulaires** : `<label>` explicite ou `aria-label` sur chaque input — jamais de placeholder seul
- **États de chargement** : `aria-busy`, spinners avec `role="status"` + texte pour lecteurs d'écran
- **Statut offline** : `aria-live="polite"` sur la bannière offline, icônes toujours accompagnées de texte accessible
- **Traducteur audio** : bouton lecture avec `aria-label` descriptif, contrôle volume accessible
- **Couleur seule** : le statut des voyages n'est jamais indiqué uniquement par la couleur (badge texte + couleur)

### 9.3 Mobile & Responsive

```
Breakpoints utilisés :
- Base (mobile first) : < 640px — layout colonne unique, bottom navigation
- sm  : 640px          — padding augmenté
- md  : 768px          — grille 2 colonnes sur le dashboard
- lg  : 1024px         — sidebar possible (futur back-office)

Cibles touch :
- Taille minimale : 44×44px (WCAG 2.5.8)
- Espacement entre cibles tactiles : ≥ 8px
- Bottom navigation : hauteur 64px + safe area inset (iOS notch)

Safe areas :
- Utiliser padding-bottom: env(safe-area-inset-bottom) sur la nav
- Tester sur iPhone SE (375px) et iPhone 15 Pro (393px)
```

### 9.4 Offline — règles d'implémentation

| Donnée | Stratégie |
|---|---|
| Itinéraire du voyage actif | Cache React Query (`gcTime` 24h) + Zustand persist |
| Documents PDF/images | Service Worker Cache API (cache-first) après premier chargement |
| Taux de change | Dernier taux connu affiché avec date de mise à jour |
| Traducteur | Désactivé offline avec message explicatif — pas d'erreur générique |
| Ajout/édition étape | Bloqué offline avec toast explicatif (pas de queue de mutation en V1) |
| Souvenirs (ajout) | Bloqué offline — upload nécessite connexion |

> **Principe** : Ce qui est indisponible offline est visuellement désactivé (opacité + cursor-not-allowed + tooltip explicatif). Jamais d'erreur 500 visible pour l'utilisateur.

### 9.5 Internationalisation (i18n)

Bien que le MVP soit en français, anticiper l'i18n dès le début :
- Utiliser `next-intl` avec namespace par module
- Toutes les chaînes dans `/messages/fr.json` — zéro chaîne en dur dans les composants
- Formateurs de dates/montants via `Intl.DateTimeFormat` et `Intl.NumberFormat` (respecte la locale automatiquement)
- Le `formatCurrency` du CurrencyConverter est déjà implémenté avec `Intl.NumberFormat`

---

## 10. ANNEXE — TYPES TYPESCRIPT PARTAGÉS

```ts
// src/types/trip.ts

export type TripStatus = 'upcoming' | 'active' | 'past'

export interface Trip {
  id: string
  name: string
  destination: string
  startDate: string          // ISO 8601
  endDate: string            // ISO 8601
  status: TripStatus
  coverImageUrl?: string
  participantCount: number
  currency: string           // code ISO 4217 ex. "EUR"
  locale: string             // ex. "ja" pour le traducteur
  timezone: string           // ex. "Asia/Tokyo"
  createdAt: string
}

export interface Step {
  id: string
  tripId: string
  title: string
  category: 'transport' | 'accommodation' | 'activity' | 'restaurant' | 'other'
  startTime: string          // ISO 8601 avec timezone
  endTime?: string
  location?: {
    name: string
    address?: string
    lat?: number
    lng?: number
  }
  notes?: string
  documentIds: string[]
  createdAt: string
}

export interface Document {
  id: string
  tripId: string
  stepId?: string
  name: string
  category: 'flight' | 'accommodation' | 'activity' | 'other'
  fileType: 'pdf' | 'jpg' | 'png'
  fileSize: number           // en octets
  url: string                // URL signée (expirable)
  thumbnailUrl?: string
  createdAt: string
}

export interface Memory {
  id: string
  tripId: string
  type: 'photo' | 'note'
  imageUrl?: string
  caption?: string
  location?: string
  takenAt?: string           // ISO 8601
  createdAt: string
}

export interface Participant {
  id: string
  tripId: string
  userId?: string            // null si invité non inscrit
  email: string
  displayName?: string
  avatarUrl?: string
  role: 'admin' | 'member' | 'viewer'
  status: 'accepted' | 'pending'
}
```

---

## 11

— FRONTEND ENGINEER

## 11. CHECKLIST DE LIVRAISON — COMPOSANTS CLÉS

> Suite et fin du livrable Frontend Engineer — Travel Companion App

---

### 11.1 Checklist par composant implémenté

| Composant | Fichier | Statut | Notes |
|---|---|---|---|
| `TripCard` | `components/trips/TripCard.tsx` | ✅ Spécifié | Carte voyage dashboard |
| `ItineraryDayView` | `components/itinerary/ItineraryDayView.tsx` | ✅ Spécifié | Vue jour par jour |
| `CurrencyConverter` | `components/tools/CurrencyConverter.tsx` | ✅ Implémenté | Debounce + Intl |
| `OfflineBanner` | `components/ui/OfflineBanner.tsx` | ✅ Implémenté | aria-live |
| `DocumentCard` | `components/documents/DocumentCard.tsx` | À implémenter | Pattern fourni |
| `MemoryGallery` | `components/memories/MemoryGallery.tsx` | À implémenter | Pattern fourni |
| `TranslatorPanel` | `components/tools/TranslatorPanel.tsx` | ✅ Implémenté | Audio + offline |
| `StepDetail` | `components/itinerary/StepDetail.tsx` | À implémenter | Pattern fourni |
| `TripCreateForm` | `components/trips/TripCreateForm.tsx` | À implémenter | Pattern fourni |
| `ParticipantList` | `components/participants/ParticipantList.tsx` | À implémenter | Pattern fourni |

---

## 12. COMPOSANTS CLÉS — CODE CIBLÉ

Les composants suivants sont les plus représentatifs de l'architecture. Ils couvrent les patterns les plus complexes : état asynchrone, offline, accessibilité, contextualisation voyage actif.

---

### 12.1 `TripCard` — Carte voyage du dashboard

```tsx
// src/components/trips/TripCard.tsx
'use client'

import { memo } from 'react'
import Image from 'next/image'
import Link from 'next/link'
import { MapPin, Calendar, Users } from 'lucide-react'
import { cn } from '@/lib/utils'
import type { Trip, TripStatus } from '@/types/trip'

// --- Helpers locaux ---

const STATUS_CONFIG: Record<TripStatus, { label: string; className: string }> = {
  upcoming: {
    label: 'À venir',
    className: 'bg-blue-100 text-blue-700',
  },
  active: {
    label: 'En cours',
    className: 'bg-emerald-100 text-emerald-700',
  },
  past: {
    label: 'Passé',
    className: 'bg-neutral-100 text-neutral-500',
  },
}

function formatDateRange(startDate: string, endDate: string): string {
  const fmt = new Intl.DateTimeFormat('fr-FR', { day: 'numeric', month: 'short' })
  return `${fmt.format(new Date(startDate))} → ${fmt.format(new Date(endDate))}`
}

// --- Composant ---

interface TripCardProps {
  trip: Trip
  /** Mode compact pour la liste secondaire (voyages passés / à venir) */
  compact?: boolean
  className?: string
}

export const TripCard = memo(function TripCard({
  trip,
  compact = false,
  className,
}: TripCardProps) {
  const status = STATUS_CONFIG[trip.status]

  return (
    <Link
      href={`/trips/${trip.id}`}
      className={cn(
        'group relative flex flex-col overflow-hidden rounded-2xl bg-white shadow-sm',
        'border border-neutral-100',
        'transition-all duration-200',
        'hover:shadow-md hover:-translate-y-0.5',
        // Focus visible accessible — jamais supprimé
        'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500 focus-visible:ring-offset-2',
        compact ? 'flex-row items-center gap-3 p-3' : 'flex-col',
        className
      )}
      aria-label={`Voyage ${trip.name} — ${status.label}`}
    >
      {/* Image de couverture */}
      <div
        className={cn(
          'relative shrink-0 bg-neutral-200',
          compact ? 'h-16 w-16 rounded-xl' : 'h-40 w-full'
        )}
      >
        {trip.coverImageUrl ? (
          <Image
            src={trip.coverImageUrl}
            alt={`Photo de couverture du voyage ${trip.name}`}
            fill
            className="object-cover"
            // Tailles adaptatives selon le layout
            sizes={compact ? '64px' : '(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw'}
          />
        ) : (
          // Placeholder avec initiale de destination
          <div className="flex h-full w-full items-center justify-center">
            <MapPin className="h-8 w-8 text-neutral-400" aria-hidden />
          </div>
        )}

        {/* Badge statut — couleur + texte (jamais couleur seule) */}
        {!compact && (
          <span
            className={cn(
              'absolute top-3 left-3 rounded-full px-2.5 py-1 text-xs font-semibold',
              status.className
            )}
            // Le texte du badge est déjà dans l'aria-label du lien
            aria-hidden
          >
            {status.label}
          </span>
        )}
      </div>

      {/* Contenu textuel */}
      <div className={cn('flex flex-col gap-1', compact ? 'flex-1 min-w-0' : 'p-4')}>
        <div className="flex items-start justify-between gap-2">
          <h3
            className={cn(
              'font-semibold text-neutral-900 leading-tight',
              compact ? 'text-sm truncate' : 'text-base'
            )}
          >
            {trip.name}
          </h3>
          {/* Badge statut en mode compact */}
          {compact && (
            <span
              className={cn(
                'shrink-0 rounded-full px-2 py-0.5 text-xs font-medium',
                status.className
              )}
              aria-hidden
            >
              {status.label}
            </span>
          )}
        </div>

        <p className={cn('flex items-center gap-1 text-neutral-500', compact ? 'text-xs' : 'text-sm')}>
          <MapPin className="h-3.5 w-3.5 shrink-0" aria-hidden />
          <span className="truncate">{trip.destination}</span>
        </p>

        <p className={cn('flex items-center gap-1 text-neutral-500', compact ? 'text-xs' : 'text-sm')}>
          <Calendar className="h-3.5 w-3.5 shrink-0" aria-hidden />
          <time dateTime={`${trip.startDate}/${trip.endDate}`}>
            {formatDateRange(trip.startDate, trip.endDate)}
          </time>
        </p>

        {/* Participants — uniquement en mode normal */}
        {!compact && trip.participantCount > 1 && (
          <p className="flex items-center gap-1 text-sm text-neutral-500">
            <Users className="h-3.5 w-3.5 shrink-0" aria-hidden />
            <span>{trip.participantCount} voyageurs</span>
          </p>
        )}
      </div>
    </Link>
  )
})

TripCard.displayName = 'TripCard'
```

---

### 12.2 `ItineraryDayView` — Vue itinéraire par jour

```tsx
// src/components/itinerary/ItineraryDayView.tsx
'use client'

import { useState, useId } from 'react'
import { ChevronDown, Plane, Hotel, Ticket, UtensilsCrossed, Circle, Clock, MapPin, FileText } from 'lucide-react'
import { cn } from '@/lib/utils'
import { useOfflineStore } from '@/store/offlineStore'
import type { Step } from '@/types/trip'

// --- Configuration des catégories ---

const CATEGORY_CONFIG = {
  transport: { label: 'Transport', Icon: Plane,            color: 'text-blue-600 bg-blue-50' },
  accommodation: { label: 'Hébergement', Icon: Hotel,     color: 'text-purple-600 bg-purple-50' },
  activity: { label: 'Activité', Icon: Ticket,            color: 'text-amber-600 bg-amber-50' },
  restaurant: { label: 'Restaurant', Icon: UtensilsCrossed, color: 'text-rose-600 bg-rose-50' },
  other: { label: 'Autre', Icon: Circle,                  color: 'text-neutral-600 bg-neutral-50' },
} as const

// --- Sous-composant : une étape individuelle ---

interface StepItemProps {
  step: Step
  isLast: boolean
}

function StepItem({ step, isLast }: StepItemProps) {
  const [isExpanded, setIsExpanded] = useState(false)
  const contentId = useId()
  const isOffline = useOfflineStore(s => s.isOffline)

  const { Icon, color, label } = CATEGORY_CONFIG[step.category]

  const formatTime = (iso: string) =>
    new Intl.DateTimeFormat('fr-FR', {
      hour: '2-digit',
      minute: '2-digit',
      timeZone: step.location ? undefined : 'UTC',
    }).format(new Date(iso))

  return (
    <li className="relative flex gap-4">
      {/* Ligne verticale de timeline */}
      {!isLast && (
        <div
          className="absolute left-[22px] top-10 bottom-0 w-0.5 bg-neutral-200"
          aria-hidden
        />
      )}

      {/* Icône catégorie */}
      <div
        className={cn(
          'relative z-10 flex h-11 w-11 shrink-0 items-center justify-center rounded-full',
          color
        )}
        aria-hidden
      >
        <Icon className="h-5 w-5" />
      </div>

      {/* Contenu de l'étape */}
      <div className="flex-1 pb-6">
        <button
          type="button"
          onClick={() => setIsExpanded(prev => !prev)}
          aria-expanded={isExpanded}
          aria-controls={contentId}
          className={cn(
            'w-full text-left rounded-xl border border-neutral-100 bg-white p-3',
            'transition-shadow duration-150 hover:shadow-sm',
            'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500 focus-visible:ring-offset-1'
          )}
        >
          <div className="flex items-start justify-between gap-2">
            <div className="flex-1 min-w-0">
              {/* Heure */}
              <p className="flex items-center gap-1 text-xs text-neutral-400 mb-0.5">
                <Clock className="h-3 w-3" aria-hidden />
                <time dateTime={step.startTime}>{formatTime(step.startTime)}</time>
                {step.endTime && (
                  <>
                    <span aria-hidden>→</span>
                    <time dateTime={step.endTime}>{formatTime(step.endTime)}</time>
                  </>
                )}
              </p>

              {/* Titre */}
              <h3 className="font-semibold text-neutral-900 text-sm leading-snug truncate">
                {step.title}
              </h3>

              {/* Lieu */}
              {step.location && (
                <p className="flex items-center gap-1 text-xs text-neutral-500 mt-0.5 truncate">
                  <MapPin className="h-3 w-3 shrink-0" aria-hidden />
                  {step.location.name}
                </p>
              )}
            </div>

            {/* Badge catégorie + chevron */}
            <div className="flex items-center gap-2 shrink-0">
              <span className="hidden sm:block text-xs text-neutral-400">{label}</span>
              <ChevronDown
                className={cn(
                  'h-4 w-4 text-neutral-400 transition-transform duration-200',
                  isExpanded && 'rotate-180'
                )}
                aria-hidden
              />
            </div>
          </div>
        </button>

        {/* Panneau expandable — notes + documents */}
        <div
          id={contentId}
          role="region"
          aria-label={`Détails de l'étape ${step.title}`}
          className={cn(
            'overflow-hidden transition-all duration-200',
            isExpanded ? 'max-h-96 opacity-100 mt-2' : 'max-h-0 opacity-0'
          )}
        >
          <div className="rounded-xl border border-neutral-100 bg-neutral-50 p-3 space-y-3">
            {/* Notes */}
            {step.notes && (
              <p className="text-sm text-neutral-700 leading-relaxed">{step.notes}</p>
            )}

            {/* Documents associés */}
            {step.documentIds.length > 0 && (
              <div>
                <p className="text-xs font-medium text-neutral-500 mb-1.5 flex items-center gap-1">
                  <FileText className="h-3.5 w-3.5" aria-hidden />
                  Documents ({step.documentIds.length})
                </p>
                <ul className="flex flex-wrap gap-2" aria-label="Documents associés">
                  {step.documentIds.map(docId => (
                    <li key={docId}>
                      {/* DocumentChip — composant minimal à implémenter */}
                      <a
                        href={`/documents/${docId}`}
                        className={cn(
                          'inline-flex items-center gap-1 rounded-lg px-2.5 py-1',
                          'text-xs font-medium bg-white border border-neutral-200',
                          'hover:border-primary-300 transition-colors',
                          'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500',
                          // Désactivé offline si le document n'est pas en cache
                          isOffline && 'opacity-50 pointer-events-none'
                        )}
                        aria-disabled={isOffline}
                        aria-label={isOffline ? `Document non disponible hors connexion` : undefined}
                      >
                        <FileText className="h-3 w-3 text-neutral-400" aria-hidden />
                        Document
                      </a>
                    </li>
                  ))}
                </ul>
              </div>
            )}

            {/* Bouton Maps — désactivé offline */}
            {step.location?.address && (
              <a
                href={
                  isOffline
                    ? undefined
                    : `https://maps.google.com/?q=${encodeURIComponent(step

— FRONTEND ENGINEER

*(Continuation exacte depuis le dernier mot produit)*

```tsx
                    href={
                      isOffline
                        ? undefined
                        : `https://maps.google.com/?q=${encodeURIComponent(step.location.address)}`
                    }
                    target="_blank"
                    rel="noopener noreferrer"
                    className={cn(
                      'inline-flex items-center gap-1.5 rounded-lg px-3 py-1.5',
                      'text-xs font-medium text-white bg-primary-600',
                      'hover:bg-primary-700 transition-colors',
                      'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500 focus-visible:ring-offset-1',
                      isOffline && 'opacity-40 pointer-events-none'
                    )}
                    aria-disabled={isOffline}
                    aria-label={
                      isOffline
                        ? 'Maps non disponible hors connexion'
                        : `Ouvrir ${step.location.name} dans Google Maps`
                    }
                  >
                    <MapPin className="h-3.5 w-3.5" aria-hidden />
                    Voir sur Maps
                  </a>
                )}
              </div>
            </div>
          </div>
        </li>
      )
    }

    // --- Composant principal : ItineraryDayView ---

    interface ItineraryDayViewProps {
      date: string       // ISO date : "2025-07-14"
      steps: Step[]
    }

    export function ItineraryDayView({ date, steps }: ItineraryDayViewProps) {
      const formattedDate = new Intl.DateTimeFormat('fr-FR', {
        weekday: 'long',
        day: 'numeric',
        month: 'long',
      }).format(new Date(date))

      // Tri par heure de début, étapes sans heure en dernier
      const sorted = [...steps].sort((a, b) =>
        new Date(a.startTime).getTime() - new Date(b.startTime).getTime()
      )

      return (
        <section aria-labelledby={`day-heading-${date}`}>
          <h2
            id={`day-heading-${date}`}
            className="sticky top-0 z-10 bg-neutral-50 py-2 text-sm font-semibold capitalize text-neutral-500 tracking-wide"
          >
            {formattedDate}
          </h2>

          {sorted.length === 0 ? (
            <p className="py-8 text-center text-sm text-neutral-400">
              Aucune étape prévue ce jour.
            </p>
          ) : (
            <ul
              className="mt-3 space-y-0 pl-0 list-none"
              aria-label={`Étapes du ${formattedDate}`}
            >
              {sorted.map((step, idx) => (
                <StepItem
                  key={step.id}
                  step={step}
                  isLast={idx === sorted.length - 1}
                />
              ))}
            </ul>
          )}
        </section>
      )
    }
```

---

### 12.3 `ToolsDrawer` — Hub outils contextuel persistant

```tsx
// src/components/tools/ToolsDrawer.tsx
'use client'

import { useRef, useEffect, useCallback } from 'react'
import { X, Languages, DollarSign, Clock } from 'lucide-react'
import { cn } from '@/lib/utils'
import { useTripStore } from '@/store/tripStore'
import { TranslatorPanel } from './TranslatorPanel'
import { CurrencyPanel } from './CurrencyPanel'
import { TimezonePanel } from './TimezonePanel'
import { useToolsDrawer } from '@/store/uiStore'

// Les trois outils disponibles
const TOOLS = [
  { id: 'translator',  label: 'Traducteur',  Icon: Languages },
  { id: 'currency',    label: 'Devises',      Icon: DollarSign },
  { id: 'timezone',    label: 'Horaires',     Icon: Clock },
] as const

type ToolId = (typeof TOOLS)[number]['id']

export function ToolsDrawer() {
  const { isOpen, activeToolId, openTool, closeTool } = useToolsDrawer()
  const drawerRef = useRef<HTMLDivElement>(null)
  const closeButtonRef = useRef<HTMLButtonElement>(null)
  const activeTrip = useTripStore(s => s.activeTrip)

  // --- Gestion focus trap et fermeture clavier ---
  useEffect(() => {
    if (!isOpen) return

    // Focus sur le bouton de fermeture à l'ouverture
    closeButtonRef.current?.focus()

    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape') closeTool()

      // Focus trap basique
      if (e.key === 'Tab' && drawerRef.current) {
        const focusable = drawerRef.current.querySelectorAll<HTMLElement>(
          'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
        )
        const first = focusable[0]
        const last = focusable[focusable.length - 1]

        if (e.shiftKey && document.activeElement === first) {
          e.preventDefault()
          last.focus()
        } else if (!e.shiftKey && document.activeElement === last) {
          e.preventDefault()
          first.focus()
        }
      }
    }

    document.addEventListener('keydown', handleKeyDown)
    return () => document.removeEventListener('keydown', handleKeyDown)
  }, [isOpen, closeTool])

  // --- Rendu du panneau actif ---
  const renderActivePanel = useCallback((toolId: ToolId) => {
    // Le contexte voyage actif est injecté automatiquement dans chaque panneau
    const context = {
      destinationLang: activeTrip?.destinationLanguageCode ?? 'en',
      destinationCurrency: activeTrip?.currency ?? 'USD',
      destinationTimezone: activeTrip?.timezone ?? 'UTC',
      originCurrency: 'EUR', // depuis le profil utilisateur
    }

    switch (toolId) {
      case 'translator': return <TranslatorPanel context={context} />
      case 'currency':   return <CurrencyPanel context={context} />
      case 'timezone':   return <TimezonePanel context={context} />
    }
  }, [activeTrip])

  return (
    <>
      {/* Backdrop */}
      <div
        className={cn(
          'fixed inset-0 z-40 bg-black/40 backdrop-blur-sm transition-opacity duration-300',
          isOpen ? 'opacity-100 pointer-events-auto' : 'opacity-0 pointer-events-none'
        )}
        onClick={closeTool}
        aria-hidden="true"
      />

      {/* Drawer — slide depuis le bas */}
      <div
        ref={drawerRef}
        role="dialog"
        aria-modal="true"
        aria-label="Outils de voyage"
        className={cn(
          'fixed bottom-0 left-0 right-0 z-50 flex flex-col',
          'rounded-t-2xl bg-white shadow-2xl',
          'transition-transform duration-300 ease-out',
          'max-h-[85vh]',
          isOpen ? 'translate-y-0' : 'translate-y-full'
        )}
      >
        {/* Drag handle */}
        <div className="flex justify-center pt-3 pb-1" aria-hidden>
          <div className="h-1 w-10 rounded-full bg-neutral-200" />
        </div>

        {/* Header */}
        <div className="flex items-center justify-between px-4 pb-3 pt-1 border-b border-neutral-100">
          <h2 className="text-base font-semibold text-neutral-900">Outils</h2>
          <button
            ref={closeButtonRef}
            type="button"
            onClick={closeTool}
            className={cn(
              'flex h-8 w-8 items-center justify-center rounded-full',
              'text-neutral-400 hover:bg-neutral-100 hover:text-neutral-700',
              'transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500'
            )}
            aria-label="Fermer les outils"
          >
            <X className="h-4 w-4" aria-hidden />
          </button>
        </div>

        {/* Onglets outils */}
        <div
          role="tablist"
          aria-label="Sélection de l'outil"
          className="flex gap-1 px-4 pt-3 pb-2"
        >
          {TOOLS.map(({ id, label, Icon }) => (
            <button
              key={id}
              role="tab"
              type="button"
              aria-selected={activeToolId === id}
              aria-controls={`panel-${id}`}
              id={`tab-${id}`}
              onClick={() => openTool(id)}
              className={cn(
                'flex flex-1 items-center justify-center gap-1.5 rounded-xl py-2 text-xs font-medium',
                'transition-all duration-150',
                'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500 focus-visible:ring-offset-1',
                activeToolId === id
                  ? 'bg-primary-600 text-white shadow-sm'
                  : 'text-neutral-500 hover:bg-neutral-100'
              )}
            >
              <Icon className="h-3.5 w-3.5" aria-hidden />
              <span>{label}</span>
            </button>
          ))}
        </div>

        {/* Panneau actif */}
        <div
          id={`panel-${activeToolId}`}
          role="tabpanel"
          aria-labelledby={`tab-${activeToolId}`}
          className="flex-1 overflow-y-auto px-4 pb-6 pt-2"
        >
          {activeToolId && renderActivePanel(activeToolId)}
        </div>
      </div>
    </>
  )
}
```

---

### 12.4 `CurrencyPanel` — Convertisseur de devises contextuel

```tsx
// src/components/tools/CurrencyPanel.tsx
'use client'

import { useState, useTransition, useEffect } from 'react'
import { ArrowLeftRight, WifiOff, RefreshCw } from 'lucide-react'
import { cn } from '@/lib/utils'
import { useOfflineStore } from '@/store/offlineStore'
import { useCurrencyRates } from '@/hooks/useCurrencyRates'

interface CurrencyContext {
  originCurrency: string
  destinationCurrency: string
}

interface CurrencyPanelProps {
  context: CurrencyContext
}

// Formatage monétaire localisé
function formatAmount(amount: number, currency: string) {
  return new Intl.NumberFormat('fr-FR', {
    style: 'currency',
    currency,
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(amount)
}

export function CurrencyPanel({ context }: CurrencyPanelProps) {
  const [amount, setAmount] = useState<string>('1')
  const [isReversed, setIsReversed] = useState(false)
  const [isPending, startTransition] = useTransition()
  const isOffline = useOfflineStore(s => s.isOffline)

  // Devises selon sens de conversion
  const fromCurrency = isReversed ? context.destinationCurrency : context.originCurrency
  const toCurrency   = isReversed ? context.originCurrency      : context.destinationCurrency

  // Hook de récupération des taux — gère le cache Redis côté API
  const { rate, lastUpdated, isLoading, refetch } = useCurrencyRates(fromCurrency, toCurrency)

  // Calcul du résultat
  const numericAmount = parseFloat(amount.replace(',', '.')) || 0
  const convertedAmount = rate !== null ? numericAmount * rate : null

  // Formatage de la date de mise à jour des taux
  const formattedLastUpdated = lastUpdated
    ? new Intl.DateTimeFormat('fr-FR', {
        day: 'numeric', month: 'short', hour: '2-digit', minute: '2-digit',
      }).format(new Date(lastUpdated))
    : null

  return (
    <div className="space-y-4">
      {/* Indicateur offline */}
      {isOffline && (
        <div
          role="status"
          aria-live="polite"
          className="flex items-center gap-2 rounded-xl bg-amber-50 px-3 py-2 text-xs text-amber-700"
        >
          <WifiOff className="h-3.5 w-3.5 shrink-0" aria-hidden />
          <span>Taux en cache — peut ne pas être à jour</span>
        </div>
      )}

      {/* Champ montant source */}
      <div>
        <label
          htmlFor="currency-amount"
          className="mb-1.5 block text-xs font-medium text-neutral-500"
        >
          Montant en {fromCurrency}
        </label>
        <div className="relative">
          <input
            id="currency-amount"
            type="number"
            inputMode="decimal"
            min="0"
            step="0.01"
            value={amount}
            onChange={e => setAmount(e.target.value)}
            placeholder="0.00"
            className={cn(
              'w-full rounded-xl border border-neutral-200 bg-white px-4 py-3',
              'text-lg font-semibold text-neutral-900',
              'placeholder:text-neutral-300',
              'focus:border-primary-400 focus:outline-none focus:ring-2 focus:ring-primary-500/20',
              'transition-colors'
            )}
            aria-label={`Montant à convertir en ${fromCurrency}`}
          />
          <span
            className="absolute right-4 top-1/2 -translate-y-1/2 text-sm font-medium text-neutral-400"
            aria-hidden
          >
            {fromCurrency}
          </span>
        </div>
      </div>

      {/* Bouton inversion des devises */}
      <div className="flex items-center justify-center">
        <button
          type="button"
          onClick={() => startTransition(() => setIsReversed(prev => !prev))}
          className={cn(
            'flex h-10 w-10 items-center justify-center rounded-full',
            'border-2 border-neutral-200 bg-white text-neutral-500',
            'hover:border-primary-400 hover:text-primary-600',
            'transition-all duration-200',
            'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500',
            isPending && 'animate-spin'
          )}
          aria-label={`Inverser : convertir depuis ${toCurrency} vers ${fromCurrency}`}
        >
          <ArrowLeftRight className="h-4 w-4" aria-hidden />
        </button>
      </div>

      {/* Résultat de

— FRONTEND ENGINEER

*(Suite et fin du livrable)*

---

```tsx
      {/* Résultat de la conversion */}
      <div
        className={cn(
          'rounded-2xl border-2 p-4 text-center transition-colors',
          convertedAmount !== null
            ? 'border-primary-100 bg-primary-50'
            : 'border-neutral-100 bg-neutral-50'
        )}
        aria-live="polite"
        aria-atomic="true"
      >
        {isLoading ? (
          <div className="flex items-center justify-center gap-2 text-sm text-neutral-400">
            <RefreshCw className="h-4 w-4 animate-spin" aria-hidden />
            <span>Chargement du taux…</span>
          </div>
        ) : convertedAmount !== null ? (
          <>
            <p className="text-xs font-medium uppercase tracking-wide text-neutral-400 mb-1">
              Équivalent en {toCurrency}
            </p>
            <p className="text-3xl font-bold text-primary-700" aria-label={`${formatAmount(convertedAmount, toCurrency)}`}>
              {formatAmount(convertedAmount, toCurrency)}
            </p>
            {rate !== null && (
              <p className="mt-2 text-xs text-neutral-400">
                1 {fromCurrency} = {rate.toFixed(4)} {toCurrency}
                {formattedLastUpdated && (
                  <span className="ml-1 text-neutral-300">· mis à jour {formattedLastUpdated}</span>
                )}
              </p>
            )}
          </>
        ) : (
          <p className="text-sm text-neutral-400">
            {isOffline ? 'Aucun taux en cache disponible' : 'Saisissez un montant pour convertir'}
          </p>
        )}
      </div>

      {/* Bouton refresh manuel */}
      {!isOffline && (
        <button
          type="button"
          onClick={() => refetch()}
          className={cn(
            'flex w-full items-center justify-center gap-2 rounded-xl py-2',
            'text-xs font-medium text-neutral-400',
            'hover:text-primary-600 transition-colors',
            'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-primary-500'
          )}
          aria-label="Actualiser le taux de change"
        >
          <RefreshCw className="h-3.5 w-3.5" aria-hidden />
          Actualiser le taux
        </button>
      )}
    </div>
  )
}
```

---

## 13. HOOKS PERSONNALISÉS — LES PLUS CRITIQUES

### 13.1 `useActiveTrip` — Source de vérité du voyage en cours

```tsx
// src/hooks/useActiveTrip.ts
import { useQuery } from '@tanstack/react-query'
import { useTripStore } from '@/store/tripStore'
import { tripsApi } from '@/lib/api/trips'
import type { Trip } from '@/types/trip'

/**
 * Récupère le voyage actif depuis le store local (optimiste)
 * puis le synchronise avec l'API si en ligne.
 * Expose un objet enrichi avec les helpers de contexte utilisés par les outils.
 */
export function useActiveTrip() {
  const activeTripId = useTripStore(s => s.activeTripId)
  const setActiveTrip = useTripStore(s => s.setActiveTrip)

  const query = useQuery<Trip>({
    queryKey: ['trip', activeTripId],
    queryFn: () => tripsApi.getById(activeTripId!),
    enabled: !!activeTripId,
    staleTime: 5 * 60 * 1000, // 5 minutes — évite les requêtes répétées en usage normal
    gcTime: 24 * 60 * 60 * 1000, // 24h — maintient en cache pour l'offline
    onSuccess: (data) => setActiveTrip(data), // synchro store local
  })

  const trip = query.data

  return {
    ...query,
    activeTrip: trip,
    // Helpers de contexte — utilisés par les outils utilitaires
    languageCode:          trip?.destination?.languageCode ?? 'en',
    destinationCurrency:   trip?.destination?.currency    ?? 'USD',
    destinationTimezone:   trip?.destination?.timezone    ?? 'UTC',
    destinationName:       trip?.destination?.name        ?? '',
    // Statut calculé côté client (évite une propriété calculée côté API)
    tripStatus: trip
      ? (new Date() < new Date(trip.startDate)
          ? 'upcoming'
          : new Date() > new Date(trip.endDate)
          ? 'past'
          : 'ongoing')
      : null,
  } as const
}
```

---

### 13.2 `useCurrencyRates` — Taux de change avec cache offline

```tsx
// src/hooks/useCurrencyRates.ts
import { useQuery } from '@tanstack/react-query'
import { toolsApi } from '@/lib/api/tools'

interface RatesResponse {
  rate: number
  lastUpdated: string // ISO date
}

/**
 * Récupère le taux de change entre deux devises.
 * - Le backend met en cache les taux 1h dans Redis pour limiter les appels Fixer.io
 * - Côté client, staleTime 30min pour éviter les requêtes répétées en session
 * - gcTime 24h pour maintenir le dernier taux connu hors connexion
 */
export function useCurrencyRates(from: string, to: string) {
  const query = useQuery<RatesResponse>({
    queryKey: ['currency-rate', from, to],
    queryFn: () => toolsApi.getExchangeRate(from, to),
    enabled: !!from && !!to && from !== to,
    staleTime: 30 * 60 * 1000,  // 30 min
    gcTime: 24 * 60 * 60 * 1000, // 24h
    retry: 1, // une seule tentative offline — évite le spam réseau
  })

  return {
    rate: query.data?.rate ?? null,
    lastUpdated: query.data?.lastUpdated ?? null,
    isLoading: query.isLoading,
    isError: query.isError,
    refetch: query.refetch,
  }
}
```

---

### 13.3 `useOfflineSync` — Synchronisation différée post-reconnexion

```tsx
// src/hooks/useOfflineSync.ts
import { useEffect, useRef } from 'react'
import { useQueryClient } from '@tanstack/react-query'
import { useOfflineStore } from '@/store/offlineStore'
import { useOfflineQueueStore } from '@/store/offlineQueueStore'

/**
 * Écoute la reconnexion réseau et :
 * 1. Rejoue les mutations en file d'attente (création d'étape hors ligne, etc.)
 * 2. Invalide les queries stales pour forcer un refetch
 *
 * Ce hook est monté une seule fois dans le RootLayout.
 */
export function useOfflineSync() {
  const queryClient = useQueryClient()
  const setOffline = useOfflineStore(s => s.setOffline)
  const pendingMutations = useOfflineQueueStore(s => s.queue)
  const flushQueue = useOfflineQueueStore(s => s.flush)
  const isSyncing = useRef(false)

  useEffect(() => {
    const handleOnline = async () => {
      setOffline(false)
      if (isSyncing.current || pendingMutations.length === 0) return
      isSyncing.current = true

      try {
        // Rejoue chaque mutation dans l'ordre FIFO
        await flushQueue()
        // Invalide toutes les queries voyage pour forcer la synchro
        await queryClient.invalidateQueries({ queryKey: ['trips'] })
        await queryClient.invalidateQueries({ queryKey: ['trip'] })
      } finally {
        isSyncing.current = false
      }
    }

    const handleOffline = () => setOffline(true)

    window.addEventListener('online', handleOnline)
    window.addEventListener('offline', handleOffline)

    // État initial
    if (!navigator.onLine) setOffline(true)

    return () => {
      window.removeEventListener('online', handleOnline)
      window.removeEventListener('offline', handleOffline)
    }
  }, [pendingMutations.length, flushQueue, queryClient, setOffline])
}
```

---

## 14. GESTION D'ÉTAT — STORES ZUSTAND

### 14.1 Structure des stores

```
src/store/
├── authStore.ts          # Session utilisateur, tokens
├── tripStore.ts          # Voyage actif, liste des voyages
├── offlineStore.ts       # État de connexion réseau
├── offlineQueueStore.ts  # File de mutations en attente (offline)
└── uiStore.ts            # Drawer outils ouvert/fermé, modales
```

### 14.2 `tripStore` — Store central du voyage

```tsx
// src/store/tripStore.ts
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'
import type { Trip, TripSummary } from '@/types/trip'

interface TripStore {
  // State
  activeTripId: string | null
  trips: TripSummary[]
  activeTrip: Trip | null

  // Actions
  setActiveTripId: (id: string | null) => void
  setActiveTrip: (trip: Trip) => void
  setTrips: (trips: TripSummary[]) => void
  addTrip: (trip: TripSummary) => void
  updateTrip: (id: string, patch: Partial<TripSummary>) => void
  removeTrip: (id: string) => void
  reset: () => void
}

const initialState = {
  activeTripId: null,
  trips: [],
  activeTrip: null,
}

export const useTripStore = create<TripStore>()(
  persist(
    (set) => ({
      ...initialState,

      setActiveTripId: (id) => set({ activeTripId: id }),

      setActiveTrip: (trip) => set({ activeTrip: trip }),

      setTrips: (trips) => set({ trips }),

      addTrip: (trip) =>
        set((state) => ({ trips: [trip, ...state.trips] })),

      updateTrip: (id, patch) =>
        set((state) => ({
          trips: state.trips.map((t) => (t.id === id ? { ...t, ...patch } : t)),
          activeTrip:
            state.activeTrip?.id === id
              ? { ...state.activeTrip, ...patch }
              : state.activeTrip,
        })),

      removeTrip: (id) =>
        set((state) => ({
          trips: state.trips.filter((t) => t.id !== id),
          activeTripId: state.activeTripId === id ? null : state.activeTripId,
          activeTrip: state.activeTrip?.id === id ? null : state.activeTrip,
        })),

      reset: () => set(initialState),
    }),
    {
      name: 'trip-store',
      storage: createJSONStorage(() => localStorage),
      // Ne persiste que les données légères — le détail complet reste dans React Query
      partialize: (state) => ({
        activeTripId: state.activeTripId,
        trips: state.trips,
      }),
    }
  )
)
```

---

### 14.3 `offlineQueueStore` — File de mutations hors connexion

```tsx
// src/store/offlineQueueStore.ts
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

export interface QueuedMutation {
  id: string          // UUID local
  type: string        // ex. 'CREATE_STEP' | 'UPDATE_STEP' | 'ADD_MEMORY'
  payload: unknown
  createdAt: string   // ISO — pour ordre FIFO et debug
  retries: number
}

interface OfflineQueueStore {
  queue: QueuedMutation[]
  enqueue: (mutation: Omit<QueuedMutation, 'id' | 'createdAt' | 'retries'>) => void
  dequeue: (id: string) => void
  flush: () => Promise<void>
  reset: () => void
}

export const useOfflineQueueStore = create<OfflineQueueStore>()(
  persist(
    (set, get) => ({
      queue: [],

      enqueue: (mutation) =>
        set((state) => ({
          queue: [
            ...state.queue,
            {
              ...mutation,
              id: crypto.randomUUID(),
              createdAt: new Date().toISOString(),
              retries: 0,
            },
          ],
        })),

      dequeue: (id) =>
        set((state) => ({
          queue: state.queue.filter((m) => m.id !== id),
        })),

      flush: async () => {
        const { queue, dequeue } = get()
        // Import dynamique pour éviter une dépendance circulaire
        const { mutationHandlers } = await import('@/lib/offlineMutationHandlers')

        for (const mutation of queue) {
          try {
            const handler = mutationHandlers[mutation.type]
            if (handler) {
              await handler(mutation.payload)
              dequeue(mutation.id)
            }
          } catch (err) {
            // Incrémente le compteur de retry — abandon après 3 tentatives
            set((state) => ({
              queue: state.queue.map((m) =>
                m.id === mutation.id ? { ...m, retries: m.retries + 1 } : m
              ),
            }))
            // Retire de la file après 3 échecs pour éviter le blocage
            if ((mutation.retries + 1) >= 3) dequeue(mutation.id)
          }
        }
      },

      reset: () => set({ queue: [] }),
    }),
    {
      name: 'offline-queue',
      storage: createJSONStorage(() => localStorage),
    }
  )
)
```

---

## 15. PATTERNS D'APPELS API

### 15.1 Client Axios configuré

```tsx
// src/lib/api/client.ts
import axios from 'axios'
import { useAuthStore } from '@/store/authStore'

export const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:3001/v1',
  timeout: 10_000,
  headers: { 'Content-Type': 'application/json' },
})

// Intercepteur requête — injecte le JWT
apiClient.interceptors.request.use((config) => {
  const token = useAuthStore.getState().accessToken
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})

// Intercepteur réponse — gère le refresh silencieux
let isRefreshing = false
let pendingRequests: Array<(token: string) => void> = []

apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const original = error.config

    if (error.response?.status === 401 && !original._retry) {
      original._retry = true

      if (is