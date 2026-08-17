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
| Framework | Flutter 3.32.x / Dart 3.8.x (version déjà installée en local via `fvm`) | Version stable actuelle du poste de dev. À figer dans `.fvmrc` pour que tout contributeur self-hosteur ait la même. |
| Gestion d'état | **Riverpod** (`flutter_riverpod` + `riverpod_annotation` / codegen) | DI + state management testable sans `BuildContext`, bon fit pour une couche data/repository propre. *(À valider — tu es le senior Flutter, si tu préfères Bloc ou autre, on ajuste ici et le reste du doc suit.)* |
| Navigation | **go_router** | Standard de facto, deep-linking simple (utile plus tard pour "ouvrir une série depuis une notif"), déclaratif, s'intègre bien avec Riverpod pour les redirections liées à l'auth. |
| Client HTTP | **dio** | Intercepteurs pour le JWT (attache le `Authorization: Bearer`, refresh silencieux sur 401), gestion fine des erreurs réseau, upload multipart pour l'import `.zip`. |
| Modèles / sérialisation | **freezed** + **json_serializable** | Les DTOs du backend sont typés (Swagger) ; on veut des modèles Dart immuables générés plutôt que du parsing JSON manuel, pour rester synchro avec le contrat API et détecter les breaking changes à la compilation. |
| Stockage sécurisé | **flutter_secure_storage** | Access/refresh tokens en Keychain/Keystore, jamais en `SharedPreferences` en clair. |
| Config d'environnement | `--dart-define` (`API_BASE_URL`) + fichier `env/` par flavor (dev/prod) | Un self-hosteur doit pouvoir pointer l'app sur *son* instance backend sans recompiler le code métier — l'URL de base n'est jamais codée en dur. |
| Images | `cached_network_image` | Posters/backdrops TMDB hotlinkés (jamais stockés côté backend, voir README backend) — il faut un cache client pour éviter de re-télécharger à chaque scroll. |
| Formulaires / validation | `flutter_form_builder` *(optionnel)* ou validation manuelle légère | Peu de formulaires dans le périmètre (login/register, éditer profil, créer liste) — à trancher selon préférence, pas structurant. |

## 3. Structure de dossiers

Feature-first, chaque feature reflète un module backend pour garder le mapping évident :

```
lib/
  core/
    network/          # Dio client, intercepteur JWT/refresh, exceptions réseau typées
    router/            # go_router config, guards (auth requise ou non)
    storage/            # wrapper flutter_secure_storage (tokens)
    theme/              # thème sombre + accent (voir §6), typographie, spacing
    config/            # lecture des --dart-define (API_BASE_URL, etc.)
    widgets/            # composants partagés (poster card, rating stars, empty states...)
  features/
    auth/
      data/             # AuthApi (dio), AuthRepository
      domain/           # modèles (User, AuthTokens)
      presentation/     # écrans login/register, providers Riverpod
    catalog/            # recherche + détail show/film/saison/épisode, watch-providers
    tracking/           # follow/status, watch/unwatch, rate — "mes séries", détail série
    lists/              # CRUD listes personnelles
    stats/              # écran statistiques (séries/films)
    import/              # upload export GDPR, statut du job, revue des items non matchés
  app.dart              # MaterialApp.router, thème, providers globaux
  main.dart             # entrypoint, bootstrap (config, storage)
test/
  <miroir de lib/>      # tests unitaires par feature (repositories, mapping JSON)
```

Chaque feature suit `data → domain → presentation` : `data/` parle au backend (dio) et ne connaît
que le contrat HTTP, `domain/` porte les modèles métier immuables, `presentation/` ne connaît que
Riverpod + les widgets. Une feature ne dépend jamais directement de la couche `data/` d'une autre
feature — si besoin de partage (ex: un `Show` utilisé à la fois par `catalog` et `tracking`), le
modèle vit dans `core/` ou dans le module qui en est propriétaire côté backend (ici `catalog`, qui
possède le `Show`).

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
  providers Riverpod métier ; pas de golden tests dans le périmètre v1 (coût d'entretien élevé
  pour un projet solo/communautaire).
- Nommage fichiers : `snake_case.dart`, un fichier = une classe publique principale.
- i18n : l'app originale est en français dans les captures (utilisateur FR) ; prévoir
  `flutter_localizations` dès le départ (`fr` + `en`) plutôt que du texte en dur, pour rester
  cohérent avec l'esprit "self-hostable par n'importe qui".

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
   notamment §2 (Riverpod vs autre) et §7 (lint strict ou non).
2. Une fois validé, on scaffolde le projet (`flutter create`, arborescence §3, packages §2) et on
   génère les modèles Dart à partir du contrat API exact du backend (routes + DTOs détaillés,
   au-delà du résumé du §4).
3. Aucune commande Flutter ni fichier de code n'est créé avant cette validation.
