# TVTime Clone — App Flutter

Document de conception, à valider avant tout code. Rien n'est implémenté tant que ce fichier
n'est pas approuvé — voir [Prochaines étapes](#prochaines-étapes).

## 1. Vue d'ensemble

App mobile de tracking personnel de séries/films (remplaçante de TV Time, fermé). Elle consomme
l'API du backend self-hosté [`another_tvtime_backend`](../another_tvtime_backend/) (NestJS +
Postgres, contrat documenté dans son [README](../another_tvtime_backend/README.md) et via Swagger
`/docs`). Aucune logique métier côté app : l'app est un client de cette API, pas une source de
vérité.

**Périmètre** (identique au backend, voir son README) : suivre des séries/films, marquer des
épisodes/films vus, noter, listes personnelles, statistiques dérivées, import d'un export GDPR
TV Time. **Explicitement hors périmètre** : tout ce qui est social (commentaires, amis, réactions,
notifications d'autres utilisateurs, badges/leaderboard) — cohérent avec le choix déjà acté côté
backend, qui n'expose de toute façon pas ces données.

**Golden rule** (héritée du backend) : minimiser les coûts d'exploitation, pérennité avant
croissance, pas de dépendance à un service tiers payant ou verrouillant. Ça s'applique aux choix
de packages ci-dessous : on préfère des libs Flutter pures, stables et maintenues plutôt que des
SDK de plateformes (pas de Firebase, pas d'Amplitude/Sentry-by-default, etc.).

## 2. Stack technique

| Domaine | Choix proposé | Pourquoi |
|---|---|---|
| Framework | **Flutter 3.47.0 / Dart 3.13.0**, géré via `fvm` (setup `fvm flutter doctor` à finaliser sur la machine de dev) | Version cible validée par l'utilisateur. À figer dans `.fvmrc` à la racine du projet Flutter pour que tout contributeur self-hosteur ait la même. |
| Gestion d'état | **Bloc/Cubit** (`flutter_bloc`, `bloc`, `equatable`) | Choix de l'utilisateur (senior Flutter, plus à l'aise avec Bloc). Séparation stricte events/states, bon fit avec une architecture en packages par feature (voir monorepo ci-dessous). |
| Monorepo / gestion des features | **Melos**, un package Dart/Flutter par feature (convention `feat_xxx`, ex. `feat_auth`, `feat_catalog`, `feat_tracking`, `feat_lists`, `feat_stats`, `feat_import`), plus deux packages transverses : **`feat_ui`** (design system — widgets purement visuels, sans logique métier, ne manipulant que des types primitifs, pas de modèles de domaine) et **`core`** (le seul endroit pour du code partagé entre plusieurs `feat_xxx` — widgets, utilitaires, ou types communs qui, sinon, provoqueraient un import cyclique entre deux features) | Choix de l'utilisateur. Isole chaque feature (deps, tests, versioning) au lieu d'un simple découpage par dossiers dans une seule app ; `melos bootstrap`/`melos run` pour orchestrer build/lint/test sur tous les packages du workspace. `feat_ui` et `core` évitent que le partage de widgets/code entre features ne redevienne un couplage direct feature-à-feature. |
| Scaffolding | **very_good_cli** (`very_good create flutter_app`, `very_good create flutter_package`) pour l'app racine et chaque package `feat_xxx` | Outil déjà installé sur la machine, choix de l'utilisateur. Génère une structure testée par défaut (lint, tests, CI templates) cohérente avec une organisation Melos. |
| Navigation | **go_router** | Standard de facto, deep-linking simple (utile plus tard pour "ouvrir une série depuis une notif"), déclaratif. |
| Client HTTP | **dio** | Intercepteurs pour le JWT (attache le `Authorization: Bearer`, refresh silencieux sur 401), gestion fine des erreurs réseau, upload multipart pour l'import `.zip`. |
| Modèles / sérialisation | **freezed** + **json_serializable** | Les DTOs du backend sont typés (Swagger) ; on veut des modèles Dart immuables générés plutôt que du parsing JSON manuel, pour rester synchro avec le contrat API et détecter les breaking changes à la compilation. |
| Stockage sécurisé | **flutter_secure_storage** | Access/refresh tokens en Keychain/Keystore, jamais en `SharedPreferences` en clair. |
| Config d'environnement | `--dart-define` (`API_BASE_URL`) + fichier `env/` par flavor (dev/prod) | Un self-hosteur doit pouvoir pointer l'app sur *son* instance backend sans recompiler le code métier — l'URL de base n'est jamais codée en dur. |
| Images | `cached_network_image` | Posters/backdrops TMDB hotlinkés (jamais stockés côté backend, voir README backend) — il faut un cache client pour éviter de re-télécharger à chaque scroll. |
| Formulaires / validation | **formz** | Choix de l'utilisateur. S'intègre naturellement avec Bloc/Cubit (un `FormzInput` par champ, validation exposée à l'état du Cubit) pour les formulaires du périmètre (login/register, éditer profil, créer liste). |
| Localisation | `flutter_localizations` + `gen-l10n`, **un fichier de traduction par package `feat_xxx`** (pas de fichier fourre-tout centralisé dans l'app) | Choix de l'utilisateur. Chaque feature reste autonome (traductions incluses), l'app agrège les délégués de localisation de chaque `feat_xxx` au lieu de posséder elle-même le texte des features. `fr` + `en` a minima (voir §7). |

### Règles de dépendance entre packages

Le monorepo Melos impose un graphe d'import strict, dans un seul sens (pas de cycle possible) :

```
                app (racine)
        ┌─────────┼─────────────────┐
        ▼         ▼                 ▼
     core     feat_xxx (auth, catalog, tracking, lists, stats, import)
        │         │  │
        └────►feat_ui◄┘
```

- **`app`** importe tous les packages (`core`, `feat_ui`, tous les `feat_xxx`) — c'est le seul endroit
  où tout est assemblé (DI, router, thème, agrégation des localisations).
- **`core`** importe uniquement `feat_ui` — jamais un `feat_xxx` (sinon cycle : un `feat_xxx` qui
  importe `core` importerait indirectement une autre feature).
- **`feat_ui`** n'importe ni `core` ni aucun `feat_xxx` — c'est le package le plus bas du graphe,
  strictement visuel (types primitifs uniquement), donc pas de raison métier d'importer quoi que ce
  soit d'autre.
- **`feat_xxx`** peut importer `core` et/ou `feat_ui`, jamais un autre `feat_xxx`, jamais `app`.
  Si deux features ont besoin de partager quelque chose, ce partagé va dans `core` (jamais un
  import direct feature → feature).

Cette règle doit être vérifiable mécaniquement (ex. lint de dépendances Melos/`custom_lint` ou CI
qui échoue si un `feat_xxx/pubspec.yaml` référence un autre `feat_xxx`), pas juste documentée —
point à préciser lors du scaffolding.

## 3. Structure de dossiers

Monorepo Melos, convention `apps/` + `packages/` (structure standard `very_good_cli`/Melos). Un
package par ligne du tableau §2 ; l'app racine assemble tout.

```
app_another_tvtime_clone/
  melos.yaml                  # scripts communs (bootstrap, analyze, test, format sur tout le repo)
  pubspec.yaml                 # workspace pub (Dart 3.13 pub workspaces)
  apps/
    tvtime/                    # app Flutter racine (scaffold `very_good create flutter_app`)
      lib/
        app/                   # App widget : MaterialApp.router, thème, agrégation des
                                # localizationsDelegates de chaque feat_xxx (voir §2)
        bootstrap.dart          # init commune (config --dart-define, storage, DI des Repository)
        main_development.dart   # entrypoints par flavor (dev/prod), cohérent very_good_cli
        main_production.dart
      test/
      pubspec.yaml               # dépend de core, feat_ui, et tous les feat_xxx
  packages/
    core/                       # scaffold `very_good create flutter_package`
      lib/
        src/
          network/               # Dio client, intercepteur JWT/refresh, exceptions typées
          router/                 # go_router config + guards (auth requise ou non)
          storage/                 # wrapper flutter_secure_storage (tokens)
          config/                  # lecture des --dart-define (API_BASE_URL, etc.)
          widgets/                  # widgets partagés à logique métier (ex: poster card qui
                                     # sait afficher un statut de tracking) — composés à partir
                                     # de feat_ui, jamais l'inverse
        core.dart                    # barrel export
      test/
      pubspec.yaml                    # dépend uniquement de feat_ui
    feat_ui/                     # scaffold `very_good create flutter_package`
      lib/
        src/
          theme/                  # thème sombre + accent (voir §6), typographie, spacing
          widgets/                 # boutons, cards, badges, empty/loading/error states —
                                    # purement visuels, types primitifs uniquement
        feat_ui.dart                # barrel export
      test/
      pubspec.yaml                  # ne dépend d'aucun autre package du repo
    feat_auth/                   # un dossier par feat_xxx, même structure interne pour tous
      lib/
        src/
          data/                    # AuthApi (dio), AuthRepository — parle au contrat HTTP
          domain/                  # modèles métier immuables (User, AuthTokens)
          presentation/             # Cubits/Blocs + écrans/widgets (login, register)
        l10n/
          arb/
            feat_auth_en.arb        # traductions propres à cette feature
            feat_auth_fr.arb
        feat_auth.dart               # barrel export (expose aussi le localizationsDelegate)
      test/
      pubspec.yaml                    # dépend de core et/ou feat_ui, jamais d'un autre feat_xxx
    feat_catalog/                # recherche + détail show/film/saison/épisode, watch-providers
    feat_tracking/                # follow/status, watch/unwatch, rate — "mes séries", détail série
    feat_lists/                    # CRUD listes personnelles
    feat_stats/                     # écran statistiques (séries/films)
    feat_import/                     # upload export GDPR, statut du job, items non matchés
```

Chaque `feat_xxx` suit `data → domain → presentation` (`data/` ne connaît que le contrat HTTP,
`domain/` porte les modèles immuables, `presentation/` ne connaît que Bloc/Cubit + les widgets de
`feat_ui`/`core`) et respecte le graphe de dépendance du §2 : jamais d'import direct vers un autre
`feat_xxx`, jamais vers `app`. Le partage inter-features passe systématiquement par `core`.

## 4. Mapping fonctionnalités ↔ endpoints backend

Basé sur `another_tvtime_backend/README.md` (préfixe `/api`, JWT Bearer requis sauf `/auth/*`).

| Feature Flutter | Endpoints backend | Écrans de référence (screenshots) |
|---|---|---|
| `auth` | `POST /auth/register`, `/login`, `/refresh`, `/logout` | Écran login/register (à designer, absent des captures — TV Time n'exportait pas cet écran) |
| `auth` (profil) | `GET/PATCH /users/me` | Onglet **Profil** (nom, avatar, stats résumées) |
| `catalog` | `GET /catalog/shows/search`, `/shows/:tmdbId`, `/shows/:tmdbId/seasons/:n`, `/shows/:tmdbId/watch-providers`, équivalents `movies/*` | Onglet **Explorer**, écran recherche ("Rechercher des séries et films", résultats avec bouton `+`) |
| `tracking` | `GET /tracking/shows`, `PATCH /tracking/shows/:tmdbId` (`isFollowing`/`isFavorite`/`isWatchlist`/`isArchived`, tous booléens indépendants — voir §5 pour le detail), watch/unwatch épisode & film, `POST /tracking/seasons/.../watch`, ratings | Onglet **Séries**/**Films** (mes séries suivies), écran détail série (bloc "Continuer le suivi", progression par saison, cases à cocher épisode), menu contextuel (Favoris, Watchlist="Regarder plus tard", Archivé="Arrêter de regarder") |
| `lists` | `GET/POST/PATCH/DELETE /lists`, items, reorder | Section **Listes** du profil ("Créer une liste") |
| `stats` | `GET /stats/me` | Écran **Statistiques** (temps passé, épisodes vus, séries ajoutées, meilleurs genres — onglets Séries/Films) |
| `import` | `POST /import` (multipart `.zip`), `GET /import/:id`, `GET/DELETE /import/unmatched` | Écran dédié dans les réglages du profil (pas dans l'app originale — spécifique à notre migration TV Time → self-hosted) |

Hors périmètre malgré leur présence dans les captures : notifications sociales, commentaires,
likes, badges, groupes, recherche d'utilisateurs — le backend n'expose aucune donnée pour ça.

## 5. Détail : statut de suivi d'une série

Le backend modélise le statut d'une série suivie avec **quatre booléens indépendants** sur
`UserShow` (`isFollowing`, `isFavorite`, `isWatchlist`, `isArchived` — voir
`another_tvtime_backend/prisma/schema.prisma`), pas un enum unique. Le menu contextuel de l'app
originale ("Favoris", "Regarder plus tard", "Arrêter de regarder") correspond à
`isFavorite`/`isWatchlist`/`isArchived` ; "Personnaliser" et "Partager" n'ont pas d'équivalent
backend (hors périmètre). Le modèle Dart `ShowTrackingStatus` doit donc rester quatre booléens,
pas un enum, pour matcher exactement le DTO `PATCH /tracking/shows/:tmdbId`.

## 6. Thème

D'après les captures d'écran originales : thème **sombre par défaut** (fond quasi noir), accents
jaune (actions principales, indicateurs de progression) et vert (statut "vu"/complété). Pas de
mode clair dans l'app originale — on peut prévoir un thème clair par simple respect des
conventions Flutter/Material 3, mais le dark reste le thème de référence et celui testé en
priorité.

## 7. Conventions

- Lint : `flutter_lints` (défaut officiel) ou `very_good_analysis` si tu préfères plus strict — à
  trancher, pas structurant.
- Tests : unitaires sur les `Repository` (mapping JSON ↔ modèle, gestion d'erreurs) et les
  Cubits/Blocs métier (`bloc_test`) ; pas de golden tests dans le périmètre v1 (coût d'entretien
  élevé pour un projet solo/communautaire).
- Nommage fichiers : `snake_case.dart`, un fichier = une classe publique principale.
- i18n : l'app originale est en français dans les captures (utilisateur FR) ; `fr` + `en` a
  minima dès le départ (jamais de texte en dur), un fichier de traduction par package `feat_xxx`
  (voir §2, ligne Localisation) — pour rester cohérent avec l'esprit "self-hostable par n'importe
  qui".

## 8. Non-objectifs explicites

- Pas de social : ni commentaires, ni amis, ni réactions, ni notifications d'autres utilisateurs,
  ni badges/leaderboard — le backend ne les expose pas, l'app ne doit pas prévoir d'UI pour ça.
- Pas de SDK de plateforme tiers par défaut (pas de Firebase Auth/Analytics/Crashlytics, pas
  d'Amplitude) — cohérent avec la golden rule coûts/pérennité. Du monitoring optionnel et
  auto-hébergeable (ex: Sentry self-hosted) pourra être ajouté plus tard, jamais comme dépendance
  dure.
- Pas de mode offline complet en v1 (pas de base locale genre Drift/Isar) — l'app suppose une
  connexion à l'instance backend. Un cache léger (`cached_network_image` pour les images, cache
  HTTP dio pour les réponses catalog) suffit pour l'expérience v1.
- Pas de paiement in-app dans le code de l'app self-hosted — la version hébergée payante
  (mentionnée dans la golden rule du projet) sera, le cas échéant, un déploiement/flavor à part,
  pas une fonctionnalité conditionnelle dans ce même code base.

## Prochaines étapes

1. Tu relis/amendes ce document directement (texte libre) jusqu'à ce qu'il te convienne —
   notamment §3 (structure à réécrire pour le monorepo Melos) et §7 (lint strict ou non).
2. Une fois validé, on scaffolde le projet (`flutter create`, arborescence §3, packages §2) et on
   génère les modèles Dart à partir du contrat API exact du backend (routes + DTOs détaillés,
   au-delà du résumé du §4).
3. Aucune commande Flutter ni fichier de code n'est créé avant cette validation.
