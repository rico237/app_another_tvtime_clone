# Playd — App Flutter (ex-« TVTime Clone »)

Document de conception, à valider avant tout code. Rien n'est implémenté tant que ce fichier
n'est pas approuvé — voir [Prochaines étapes](#prochaines-étapes).

## 1. Vue d'ensemble

App mobile de tracking personnel de séries/films (remplaçante de TV Time, fermé), publiée sous le
nom de marque **Playd** (voir `handoff/`, qui documente ce nom — l'app s'appelait "Sériothèque"
dans les toutes premières itérations de design, renommage déjà intégré). **Ce renommage concerne
uniquement l'app Flutter** (nom affiché, package `apps/playd`, valeurs par défaut) — le backend et
le nom du projet/repo (`another_tvtime_backend`, workspace `tvtime_clone`) ne changent pas, ce
sont des noms techniques internes, pas la marque publique.

Elle consomme l'API du backend self-hosté [`another_tvtime_backend`](../another_tvtime_backend/)
(NestJS + Postgres, contrat documenté dans son [README](../another_tvtime_backend/README.md) et
via Swagger `/docs`). Aucune logique métier côté app : l'app est un client de cette API, pas une
source de vérité.

**Périmètre** (identique au backend, voir son README) : suivre des séries/films, marquer des
épisodes/films vus, noter, listes personnelles, statistiques dérivées, import d'un export GDPR
TV Time. **Explicitement hors périmètre** : tout ce qui est social (commentaires, amis, réactions,
notifications d'autres utilisateurs, badges/leaderboard) — cohérent avec le choix déjà acté côté
backend, qui n'expose de toute façon pas ces données.

**Golden rule** (héritée du backend) : minimiser les coûts d'exploitation, pérennité avant
croissance, pas de dépendance à un service tiers payant ou verrouillant. Ça s'applique aux choix
de packages ci-dessous : on préfère des libs Flutter pures, stables et maintenues plutôt que des
SDK de plateformes (pas de Firebase, pas d'Amplitude/Sentry-by-default, etc.).

**Modèle de monétisation** (précisé après la revue du handoff §4) : l'app sera publiée sur les
stores (App Store / Play Store) et proposera **deux modes au choix de l'utilisateur, dans la même
app** — self-hosting gratuit (l'utilisateur pointe l'app sur son propre backend) ou un
**abonnement payant à une offre hébergée gérée** par le projet pour ceux qui ne veulent pas
s'occuper de l'aspect technique. Ce n'est **pas** un déploiement/flavor séparé : les deux options
coexistent dans `playd` **et** `template_app` (un forkeur peut proposer sa propre offre
hébergée payante via le même mécanisme). Voir §4 (feature `feat_hosting`) — **prévu
architecturalement dès maintenant, mais pas construit dans ce premier scaffolding** : ça suppose
un vrai backend multi-tenant + une intégration de paiement, un chantier à part entière.

## 2. Stack technique

| Domaine | Choix proposé | Pourquoi |
|---|---|---|
| Framework | **Flutter 3.47.0 / Dart 3.13.0**, géré via `fvm` (setup `fvm flutter doctor` à finaliser sur la machine de dev) | Version cible validée par l'utilisateur. À figer dans `.fvmrc` à la racine du projet Flutter pour que tout contributeur self-hosteur ait la même. **Plancher OS relevé par Flutter 3.47 lui-même : iOS minimum 15** (avant 13) — à répercuter dans `ios/Runner`'s deployment target au scaffolding (vérifié sur flutter.dev/blog, "What's new in Flutter 3.47"). Pas de plancher Android officiel documenté par cette version ; à vérifier au scaffolding (`android/app/build.gradle`, `minSdkVersion`) plutôt que supposer. |
| Gestion d'état | **Bloc/Cubit** (`flutter_bloc`, `bloc`, `equatable`) | Choix de l'utilisateur (senior Flutter, plus à l'aise avec Bloc). Séparation stricte events/states, bon fit avec une architecture en packages par feature (voir monorepo ci-dessous). |
| Injection de dépendances | **get_it** (service locator) | Choix de l'utilisateur. Chaque `feat_xxx` expose une fonction d'enregistrement (`registerFeatAuth(GetIt it)` — `AuthApi`, `AuthRepositoryImpl` bindée à l'interface `AuthRepository` de `domain/`, en `registerLazySingleton`), appelée depuis `bootstrap.dart` de l'app au démarrage. Les Cubits/Blocs de `presentation/` résolvent leurs dépendances via `getIt<AuthRepository>()` (jamais via un import direct de `feat_xxx`/`data`, qui reste inconnu de `presentation/`) — voir §3 et la règle de dépendance du §2. |
| Monorepo / gestion des features | **Melos**, un package Dart/Flutter par feature (convention `feat_xxx`, ex. `feat_auth`, `feat_catalog`, `feat_tracking`, `feat_lists`, `feat_stats`, `feat_import`, **`feat_hosting`** — choix du mode self-hosted/abonnement géré, voir §1 et §4 ; pour ce premier scaffolding, seule la partie self-hosted de `feat_hosting` est construite, l'abonnement/paiement est différé mais le package existe déjà pour l'accueillir sans restructuration), chacun structuré en interne avec des **dossiers** `data/`/`domain/`/`presentation/` (pas des packages séparés — choix explicite de l'utilisateur), plus trois packages transverses (le tier `shared` de l'image de référence) : **`ui_kit`** (design system — widgets purement visuels, sans logique métier, types primitifs uniquement), **`api_client`** (client HTTP : Dio, intercepteur JWT access/refresh, exceptions réseau typées — aucune logique métier, ne connaît que le contrat HTTP) et **`shared`** (router, storage, config, + widgets partagés à logique métier — le seul endroit pour du code partagé entre plusieurs `feat_xxx` qui, sinon, provoquerait un import cyclique entre deux features) | Choix de l'utilisateur, aligné sur le schéma clean architecture fourni (tier `shared` = `ui_kit` + `api_client`). Isole chaque feature (deps, tests, versioning) au lieu d'un simple découpage par dossiers dans une seule app, sans pousser jusqu'à un package par couche à l'intérieur d'une feature ; `melos bootstrap`/`melos run` pour orchestrer build/lint/test sur tous les packages du workspace. `ui_kit`/`api_client`/`shared` évitent que le partage de widgets/réseau/code entre features ne redevienne un couplage direct feature-à-feature. |
| Applications (multi-app) | Le repo héberge **deux apps** consommant les mêmes packages `feat_xxx`/`shared`/`ui_kit`/`api_client` : **`playd`** (nom de marque, ex-`tvtime_clone` — voir §1), l'instance de référence maintenue par le projet, et **`template_app`**, un squelette minimal destiné à être forké par quiconque veut créer sa propre app self-hostée à partir de ce projet | Choix de l'utilisateur : projet open source, donc prévoir dès l'architecture que d'autres personnes forkent pour créer leur propre app plutôt que de devoir extraire cette possibilité après coup. `template_app` ne contient que le strict nécessaire à personnaliser (nom, icône, accent couleur, `API_BASE_URL` par défaut) — toute la logique vit dans les packages partagés, jamais dupliquée entre apps. |
| Feature flags par app | Un objet statique par app (`AppFeatures`, ex. `apps/playd/lib/app/app_features.dart`) qui active/désactive chaque `feat_xxx` **à la compilation** — consulté par `bootstrap.dart` (n'enregistre dans get_it que les features actives, voir ligne Injection de dépendances) et par le router de `shared` (n'expose les routes/l'onglet de nav que si la feature est active). Prévoit aussi des **sous-flags à l'intérieur d'une feature** (pas seulement toute/rien) : `feat_hosting` a un sous-flag `cloudSubscriptionEnabled` (faux dans les deux apps pour ce premier scaffolding — seule la partie self-hosted du hosting-mode est construite, voir §1/§4), pour ajouter l'abonnement plus tard sans réactiver toute la feature d'un coup. | Choix de l'utilisateur. Sert directement le cas `template_app` : un forkeur peut désactiver `import` (spécifique à la migration TV Time) ou toute autre feature sans toucher au code des packages. Volontairement statique/compile-time, jamais un service de feature-flag distant (LaunchDarkly, Firebase Remote Config...) — ce serait une dépendance tierce payante/verrouillante, contraire à la golden rule du §1. **⚠️ Point à vérifier au scaffolding, pas encore acquis** : pour un vrai gain de taille de binaire (et pas juste "caché à l'écran"), le toggle doit empêcher l'**import** du code de la feature désactivée, pas seulement un `if` runtime autour d'un import déjà présent dans l'app — Dart ne tree-shake que du code jamais référencé. Il faudra valider concrètement (ex. build avec une feature désactivée + inspection de la taille de l'AOT snapshot / `flutter build apk --analyze-size`, ou génération du fichier `app_features.dart` par app plutôt qu'un flag lu à l'exécution) avant de considérer l'économie de taille acquise — sinon le flag ne fait que masquer l'UI, ce qui reste utile mais n'est pas la même promesse. |
| Scaffolding | **very_good_cli** (`very_good create flutter_app` pour chaque app, `very_good create flutter_package` pour `shared`/`ui_kit`/`api_client`/chaque `feat_xxx`) | Outil déjà installé sur la machine, choix de l'utilisateur. Génère une structure testée par défaut (lint, tests, CI templates) cohérente avec une organisation Melos. |
| Navigation | **go_router** | Standard de facto, deep-linking simple (utile plus tard pour "ouvrir une série depuis une notif"), déclaratif. |
| Client HTTP | **dio** | Intercepteurs pour le JWT (attache le `Authorization: Bearer`, refresh silencieux sur 401), gestion fine des erreurs réseau, upload multipart pour l'import `.zip`. |
| Modèles / sérialisation | **freezed** + **json_serializable** (freezed n'inclut pas json_serializable : ce sont deux packages séparés — freezed génère les classes immuables/unions/`copyWith`, json_serializable génère `fromJson`/`toJson` ; freezed s'y intègre nativement via un seul `part '*.freezed.dart'` + `part '*.g.dart'`, mais les deux dépendances restent nécessaires) | Les DTOs du backend sont typés (Swagger) ; on veut des modèles Dart immuables générés plutôt que du parsing JSON manuel, pour rester synchro avec le contrat API et détecter les breaking changes à la compilation. |
| Stockage sécurisé | **flutter_secure_storage** | Access/refresh tokens en Keychain/Keystore, jamais en `SharedPreferences` en clair. |
| Préférences locales | **shared_preferences** | Découvert via le handoff (§4), **restreint aux réglages vraiment anodins à reconfigurer** (langue, thème, mode liste/grille, lecture auto, annonces déjà vues) et **l'adresse du serveur backend** (voir ligne Config d'environnement) — jamais de secret ici, ça reste dans `flutter_secure_storage`. **Masquer les épisodes vus par titre** et **services de streaming préférés** ne sont **plus** ici : déplacés côté backend (voir §4, "Écarts") pour éviter de tout perdre à un changement d'appareil. Non synchronisé entre appareils pour ce qui reste ici (cohérent avec §8, mais l'impact d'une perte est nul pour ces items-là). |
| Config d'environnement | `--dart-define` (`API_BASE_URL`) comme valeur par défaut à la compilation, **rendue modifiable à l'exécution** via l'écran Réglages → Serveur de `feat_hosting` (persistée en `shared_preferences` par `shared`, avec un bouton "Tester la connexion" contre `GET /health`, voir le doc backend §4) | Un self-hosteur doit pouvoir pointer l'app sur *son* instance backend sans recompiler le code métier. Le handoff (§4) montre un écran dédié pour ça — mieux que `--dart-define` seul, qui impose un rebuild par instance. La plomberie (lecture/override/persistance de l'URL) vit dans `shared`, l'écran lui-même dans `feat_hosting_presentation`. **Jamais de champ clé TMDB côté app** (voir §4, écarts) — le backend gère TMDB entièrement côté serveur. |
| Flavors | **3 flavors par app** : `development` / `staging` (`stg`) / `production`, scaffoldés via `very_good_cli` (`main_development.dart`/`main_staging.dart`/`main_production.dart`), chacun avec son `API_BASE_URL` par défaut, un nom d'app et un bundle id/application id suffixés (ex. `.stg`) pour installer stg et prod côte à côte sur le même appareil | Choix de l'utilisateur. Permet de tester contre un backend de staging avant une release prod, sans affecter la prod. S'applique identiquement à `playd` et `template_app` (un forkeur peut ignorer `staging` s'il n'en a pas besoin). |
| Images | `cached_network_image` | Posters/backdrops TMDB hotlinkés (jamais stockés côté backend, voir README backend) — il faut un cache client pour éviter de re-télécharger à chaque scroll. |
| Icônes | **Lucide** (ex. `lucide_icons_flutter`) | Fixé par le handoff (§6/§4) : le design de référence utilise le jeu Lucide, tracés inline, épaisseur 1.7–2.4. |
| Nav bar liquid glass | **`adaptive_platform_ui: ^0.1.111`** (confirmé, décision finale — voir §6) : rendu natif Liquid Glass (UITabBar) sur iOS 26+, repli Cupertino sur iOS < 26, Material 3 sur Android, sélection automatique à l'exécution. `handoff/` sert de **maquette de référence** (géométrie de la capsule, courbes d'animation, comportement de compression au scroll) pour configurer/styliser le composant du package au plus près — mais le rendu final vient de la lib, pas d'un `BackdropFilter` peint à la main écran par écran. | Choix explicite de l'utilisateur : le handoff est une maquette, pas la spec finale à recopier au pixel via du code custom — le produit fini doit privilégier le **meilleur rendu natif et la meilleure couverture** sur l'ensemble des OS mobiles ciblés plutôt qu'un seul rendu peint à la main partout. Le compromis assumé : le rendu peut différer légèrement d'un OS à l'autre (matériaux natifs propres à chaque plateforme) plutôt que d'être identique au pixel près partout comme le montre la maquette. |
| Police | **Archivo**, embarquée en asset (`.ttf` dans `ui_kit/assets/fonts/`, déclarée via `fontFamily` dans le `pubspec.yaml` de `ui_kit`) — **pas** le package `google_fonts` en mode réseau | Fixé par le handoff (§6). Le package `google_fonts` télécharge les polices depuis les serveurs Google au premier lancement par défaut — contraire à l'esprit self-hosted/offline-friendly du projet (golden rule §1) ; embarquer le `.ttf` évite tout appel réseau et toute dépendance à un tiers pour afficher du texte. |
| Formulaires / validation | **formz** | Choix de l'utilisateur. S'intègre naturellement avec Bloc/Cubit (un `FormzInput` par champ, validation exposée à l'état du Cubit) pour les formulaires du périmètre (login/register, éditer profil, créer liste). |
| Splash screen | **Aucun package supplémentaire** — `CustomPainter` + `AnimationController` Flutter natifs (voir handoff `handoff/`, formules d'animation exactes fournies). Un flag de config `splashOnLaunch` (défaut `true`, désactivable — utile en tests automatisés) | Le handoff fournit les formules (vitesse de propagation, amortissement) prêtes à coder en Dart, pas de raison d'ajouter une dépendance (type Lottie/Rive) pour une animation purement géométrique — cohérent avec la golden rule (minimiser les deps). |
| Localisation | `flutter_localizations` + `gen-l10n`, **un fichier de traduction par package `feat_xxx`** (pas de fichier fourre-tout centralisé dans l'app) | Choix de l'utilisateur. Chaque feature reste autonome (traductions incluses), l'app agrège les délégués de localisation de chaque `feat_xxx` au lieu de posséder elle-même le texte des features. `fr` + `en` a minima (voir §7). |
| Documentation (README, tutoriels) | **Révisé** : pas de package de rendu Markdown, pas de docs embarquées. Chaque "carte doc" (About, futur tutoriel TMDB, futur tutoriel self-hosting) est un **`PlaydBanner`** (déjà dans le design system, voir §6) avec kicker/titre/corps écrits en dur dans `feat_hosting_presentation` (comme le fait déjà le handoff pour l'écran About) + un CTA qui ouvre l'URL GitHub **rendue** (`https://github.com/.../blob/main/docs/....md`, pas l'URL `raw.githubusercontent.com` qui affiche du Markdown brut non stylé) via **`url_launcher`** dans le navigateur système. | Choix de l'utilisateur : un vrai renderer Markdown alourdit le binaire pour un besoin ponctuel (quelques écrans de doc), alors que le texte lui-même (quelques dizaines de Ko) ne pèse rien. `url_launcher` est un package officiel, minimal, déjà quasi incontournable dans tout projet Flutter. Reproduit fidèlement ce que fait déjà le handoff (`openUrl`), sans avoir besoin d'un composant `ui_kit` dédié — `PlaydBanner` suffit, aucun nouveau composant à concevoir. |

### Règles de dépendance entre packages

Le monorepo Melos impose un graphe d'import strict, dans un seul sens (pas de cycle possible) :

```
                       playd  /  template_app
        ┌─────────────────┼─────────────────────────┐
        ▼                 ▼                          ▼
     shared           feat_xxx (auth, catalog, tracking, lists, stats, import, hosting)
        │  │              │        │        │
        │  └──────►api_client◄─────┘        │
        └───────────►ui_kit◄─────────────────┘
```

- **Une app** (`playd`, `template_app`) importe tous les packages (`shared`, `ui_kit`,
  `api_client`, tous les `feat_xxx`) — c'est le seul endroit où tout est assemblé (DI, router,
  thème, agrégation des localisations).
- **`shared`** importe `ui_kit` et `api_client` — jamais un `feat_xxx` (sinon cycle : un `feat_xxx`
  qui importe `shared` importerait indirectement une autre feature).
- **`ui_kit`** et **`api_client`** n'importent ni `shared` ni aucun `feat_xxx` — ce sont les deux
  packages les plus bas du graphe (feuilles), chacun sur son propre axe : `ui_kit` strictement
  visuel (types primitifs uniquement), `api_client` strictement HTTP (ne connaît que le contrat
  API, aucune logique métier). Ils n'ont pas de dépendance entre eux non plus.
- **`feat_xxx`** peut importer `shared`, `ui_kit` et/ou `api_client` (typiquement : `data/` importe
  `api_client`, `presentation/` importe `ui_kit`, les deux peuvent importer `shared`), jamais un
  autre `feat_xxx`, jamais une app. Si deux features ont besoin de partager quelque chose de plus
  que du HTTP/UI générique, ce partagé va dans `shared` (jamais un import direct feature →
  feature). À l'intérieur d'un `feat_xxx`, `data/` implémente les interfaces `Repository` définies
  dans `domain/`, et `presentation/` (Cubits/Blocs + écrans) ne
  dépend que de `domain/` — même règle qu'entre packages, appliquée par convention à l'intérieur
  du package puisque ce sont ici de simples dossiers, pas des unités compilées séparément.

Cette règle doit être vérifiable mécaniquement, pas juste documentée — voir §7, "Application
mécanique du graphe de dépendance" (contraintes de `pubspec.yaml` dans un Dart/pub workspace,
plutôt qu'un lint/CI qui ne détecterait la violation qu'après coup).

## 3. Structure de dossiers

Monorepo Melos, convention `apps/` + `packages/` (structure standard `very_good_cli`/Melos),
**multi-app** (§2). Un package par feature, structuré en interne (pas de package séparé par
couche).

```
app_another_tvtime_clone/
  melos.yaml                  # scripts communs (bootstrap, analyze, test, format sur tout le repo)
  pubspec.yaml                 # workspace pub (Dart 3.13 pub workspaces)
  apps/
    playd/                      # instance de référence maintenue par le projet (marque « Playd »,
                                  # dossier renommé depuis tvtime_clone — voir §1)
      lib/
        app/
          app.dart               # App widget : MaterialApp.router, thème, agrégation des
                                   # localizationsDelegates de chaque feat_xxx
          app_features.dart      # AppFeatures : quelles feat_xxx sont actives pour CETTE app
                                   # (voir §2, ligne Feature flags par app) — consulté par
                                   # bootstrap.dart et par le router de shared
        bootstrap.dart           # composition root : appelle la fonction d'enregistrement get_it
                                   # de chaque feat_xxx active dans app_features.dart
        main_development.dart    # entrypoints par flavor (dev/staging/prod), cohérent very_good_cli
        main_staging.dart
        main_production.dart
      test/
      pubspec.yaml               # dépend de shared, ui_kit, api_client, et tous les feat_xxx
    template_app/               # squelette minimal pour les forks — mêmes packages, même
                                  # bootstrap.dart/flavors ; seuls diffèrent nom/icône/accent (ui_kit
                                  # theme override), API_BASE_URL par défaut et app_features.dart
                                  # (ex. import désactivé par défaut). C'est la checklist complète de
                                  # ce qu'un forkeur doit changer pour créer sa propre app.
      lib/ ...                   # même structure que playd/
      test/
      pubspec.yaml
  packages/
    api_client/                 # scaffold `very_good create flutter_package`
      lib/
        src/
          client.dart             # Dio client, intercepteur JWT access/refresh
          exceptions/              # exceptions réseau typées (401, 404, timeout, etc.)
        api_client.dart              # barrel export
      test/
      pubspec.yaml                    # ne dépend d'aucun autre package du repo (feuille, comme ui_kit)
    ui_kit/                      # scaffold `very_good create flutter_package` — structure figée
                                   # par handoff/README.md
      lib/
        src/
          theme/
            playd_colors.dart        # PlaydColors extends ThemeExtension — tokens dark/light
                                       # (surface/border/content/brand/state/action/overlay)
            playd_typography.dart    # PlaydText.* — échelle typo complète (voir §6)
            playd_tokens.dart        # space/radius/elevation/motion
            playd_theme.dart         # PlaydTheme.dark · PlaydTheme.light
          components/               # 19 widgets nommés Playd*, purement visuels — liste complète
                                     # au §6 (PlaydButton, PlaydChip, PlaydTextField, PlaydSwitch,
                                     # PlaydGlassTabs, PlaydMediaRow, PlaydPosterTile, etc. — dont
                                     # PlaydBanner, réutilisé pour les cartes doc, voir §2/§4 : pas
                                     # de composant dédié supplémentaire pour la documentation)
          brand/
            playd_logo_mark.dart     # PlaydLogoMark({size, withWordmark}) — le "P" de 5 carrés,
                                       # toujours posé sur son carré brand.iconCanvas (voir §6)
            splash_screen.dart        # CustomPainter animation "goutte d'eau" (voir §2, §6) +
                                       # AnimationController, purement visuel/self-contained
        ui_kit.dart                  # barrel export
      test/
      pubspec.yaml                   # ne dépend d'aucun autre package du repo (feuille, comme api_client)
                                       # PAS de adaptive_platform_ui ici — la nav bar n'est
                                       # explicitement pas dans ce kit, voir shared/ ci-dessous
    shared/                      # scaffold `very_good create flutter_package`
      lib/
        src/
          router/                 # go_router config + guards (auth requise ou non)
          storage/                 # wrapper flutter_secure_storage (tokens) + wrapper
                                    # shared_preferences (prefs d'affichage, adresse serveur)
          config/                  # lecture des --dart-define (API_BASE_URL, etc.)
          navigation/
            glass_nav_bar.dart        # enrobe le tab bar de adaptive_platform_ui (voir §2), stylisé
                                        # avec les tokens de ui_kit — volontairement hors du design
                                        # system pur (dépendance à une lib externe, voir
                                        # handoff/README.md, §4)
          widgets/                  # widgets partagés à logique métier (ex: poster card qui
                                     # sait afficher un statut de tracking) — composés à partir
                                     # de ui_kit, jamais l'inverse
        shared.dart                   # barrel export
      test/
      pubspec.yaml                    # dépend de ui_kit, api_client, et adaptive_platform_ui
    feat_auth/                   # un dossier par feat_xxx, même structure interne pour tous
      lib/
        src/
          data/                    # AuthApi (via api_client), AuthRepositoryImpl — implémente domain/
          domain/                  # modèles métier immuables (User, AuthTokens) + interface
                                    # AuthRepository (abstract)
          presentation/             # Cubits/Blocs + écrans/widgets (login, register) — ne
                                    # dépend que de domain/, jamais directement de data/
        l10n/
          arb/
            feat_auth_en.arb        # traductions propres à cette feature
            feat_auth_fr.arb
        feat_auth.dart               # barrel export (expose aussi le localizationsDelegate)
      test/
      pubspec.yaml                    # dépend de shared/ui_kit/api_client selon besoin, jamais
                                        # d'un autre feat_xxx
    feat_catalog/                # recherche + détail show/film/saison/épisode, watch-providers
    feat_tracking/                # follow/status, watch/unwatch, rate — Séries ET Films (une seule
                                    # feature pour les deux, pas deux features séparées)
      lib/src/presentation/
        widgets/                    # partagés entre onglets Séries et Films : bascule liste/grille,
                                     # ligne média (PlaydMediaRow), vignette (PlaydPosterTile),
                                     # bouton "marquer vu", menu ··· — même widget, paramétré par
                                     # type d'entité (show/movie), pas deux implémentations
        show/                        # ce qui n'existe QUE pour les séries : accordéon de saisons,
                                       # carrousel "Continuer à regarder", suivi par épisode — la
                                       # seule vraie divergence entre Séries et Films (voir §5)
        movie/                        # bascule "vu" simple, pas de sous-écran saisons/épisodes
    feat_lists/                    # CRUD listes personnelles
    feat_stats/                     # écran statistiques (séries/films)
    feat_import/                     # upload export GDPR, statut du job, items non matchés
    feat_hosting/                     # choix self-hosted/cloud, adresse serveur + test connexion,
                                        # écran About/README de premier lancement (construits pour
                                        # ce scaffolding) ; écrans abonnement/paiement/factures
                                        # (différés, voir §1/§4) derrière
                                        # AppFeatures.cloudSubscriptionEnabled
```

Chaque `feat_xxx` respecte, en interne, la même direction de dépendance que le graphe du §2 :
`data/` implémente les interfaces `Repository` de `domain/` en s'appuyant sur `api_client` pour le
HTTP, `presentation/` ne dépend que de `domain/` (jamais directement de `data/` — le branchement
concret se fait dans l'app via `get_it`, voir §2). Jamais d'import direct vers un autre
`feat_xxx`, jamais vers une app. Le partage inter-features passe systématiquement par `shared`.

## 4. Mapping fonctionnalités ↔ endpoints backend

Basé sur `another_tvtime_backend/README.md` et vérifié directement contre les controllers/schema du
backend (préfixe `/api`, JWT Bearer requis sauf `/auth/*`).

**Référence visuelle** : `app_another_tvtime_clone/handoff/` — **dossier unique, évolutif**, mis à
jour au fil de l'eau plutôt que dupliqué en plusieurs handoffs versionnés. Remplace
`original_projet_screenshots/` comme source de vérité visuelle. Couvre à ce jour : tous les écrans
"app connectée" (Séries, Films, Explorer, Profil, fiches, listes, stats, réglages), la marque
**Playd** (voir §1) et son splash screen animé, le flow d'authentification complet (détaillé
ci-dessous), la nav bar **liquid glass** flottante (voir §6), et le design system consolidé pour
`ui_kit` (tokens dark+light, typo, formes, 19 composants nommés, arborescence de fichiers Flutter
proposée — largement reprise telle quelle au §3). `handoff/README.md` est la référence à lire en
premier ; `handoff/TVTime Clone.dc.html` est le prototype complet, à vérifier directement quand le
README ne suffit pas (déjà arrivé plusieurs fois — voir "Flow d'authentification" ci-dessous).

Tous "high-fidelity" (à reprendre au pixel près, sauf la nav bar — voir §6, maquette indicative
plutôt que spec littérale) et cette section fusionne leur inventaire
d'écrans avec le contrat API.

| Feature Flutter | Endpoints backend (vérifiés) | Écrans de référence (handoff) |
|---|---|---|
| `auth` | `POST /auth/register`, `/login`, `/refresh`, `/logout` | **Designés dans `handoff/`** — 7 écrans, voir "Flow d'authentification" ci-dessous |
| `auth` (profil) | `GET/PATCH /users/me` (`displayName`, `avatarUrl`, `coverUrl`, `language`, `timezone` — **pas de champ mot de passe**, voir "Écarts" ci-dessous) | Onglet **Profil**, Réglages → Compte (nom d'utilisateur, e-mail, ID) |
| `catalog` | `GET /catalog/shows/search`, `/shows/:tmdbId`, `/shows/:tmdbId/seasons/:n`, `/shows/:tmdbId/watch-providers`, équivalents `movies/*` — **pas de casting/bande-annonce/titres similaires** (voir "Écarts") | Onglet **Explorer** (recherche), fiche série/film (sections Où regarder ✅, casting ❌, bande-annonce ❌, similaires ❌) |
| `tracking` | `GET /tracking/shows`, `GET/PATCH/DELETE /tracking/shows/:tmdbId` (4 booléens indépendants, voir §5), `PUT/DELETE .../rating`, `GET/POST/DELETE` watch épisode, `POST` watch saison, `POST/DELETE` watch film, `PUT/DELETE` rating film — **pas de favori/watchlist pour les films côté backend** (voir "Écarts") | **Séries et Films sont étroitement couplés côté UI** (widgets partagés, voir §3) — seule vraie divergence : le suivi par saison/épisode, propre aux séries. Onglets **Séries**/**Films** (À regarder/À voir, À venir), fiche série (Continuer à regarder, saisons dépliables), fiche film, menu `···` (Favoris, Regarder plus tard = Watchlist, Arrêter/Reprendre = Archivé, **Supprimer** = `DELETE /tracking/shows/:tmdbId`), page dédiée **Séries préférées** (= vue filtrée `isFavorite`, backend ✅) ; **Films préférés** a le même écran côté handoff mais **rien à filtrer côté backend pour l'instant** (voir "Écarts") |
| `lists` | `GET/POST/PATCH/DELETE /lists`, items, reorder — **pas de compteur de complétions** (voir "Écarts") | Onglet Listes (index, création avec modèles, détail, mode édition drag & drop, pop-up de fin de liste) |
| `stats` | `GET /stats/me` | Écran **Statistiques** (onglets Séries/Films, activité 12 mois, répartition par genre) |
| `import` | `POST /import` (multipart `.zip`), `GET /import/:id`, `GET/DELETE /import/unmatched` | Absent du handoff (spécifique à notre migration TV Time → self-hosted, pas dans l'app originale) |
| `hosting` *(nouveau)* | `--dart-define`/config app + `GET /health` (voir doc backend) pour le self-hosted ; **pas d'endpoint backend pour l'abonnement/paiement** (différé, voir "Fonctionnalité prévue, différée" ci-dessous) | Réglages → Hébergement (mode self-hosted construit dès ce scaffolding ; mode cloud/abonnement/factures différé derrière `AppFeatures.cloudSubscriptionEnabled`) |

Hors périmètre, y compris quand présent dans le handoff (voir "Éléments à écarter" ci-dessous) :
notifications sociales, commentaires, likes, badges, groupes, recherche d'utilisateurs, sondages/
notes communautaires — le backend n'expose et n'exposera aucune donnée pour ça sans revisiter
cette décision avec toi. Le mode cloud/abonnement, en revanche, **n'est plus hors périmètre** :
voir "Fonctionnalité prévue, différée" ci-dessous.

### Flow d'authentification (7 écrans)

Couvert par `handoff/` — l'écran 0 ci-dessous n'est vérifiable que directement dans
`handoff/TVTime Clone.dc.html` (le README du handoff ne le documente pas), à garder en tête pour la
suite : le README seul ne suffit pas toujours, le prototype fait foi en cas de doute :

0. **About / "Lisez-moi"** *(nouveau, premier écran jamais affiché, une seule fois — flag persisté
   `aboutSeen`)* : un texte de mission à la première personne expliquant pourquoi Playd existe,
   plus 3 blocs (auto-hébergement = le but pas un repli, vos données restent les vôtres,
   l'abonnement paie le serveur pas des fonctionnalités), CTA "Commencer" (→ écran 1) et "Lire le
   README complet" (lien externe vers `github.com/playd-app/playd#readme`). **Bonus notable** :
   ce texte est explicitement pensé par l'utilisateur pour devenir le contenu réel du futur
   `README.md` du projet — à reprendre tel quel le moment venu, pas à réécrire de zéro.
1. **Setup / premier lancement** ("Où vivent vos données ?") — choix entre deux cartes radio :
   auto-hébergé (gratuit, adresse serveur à fournir) ou Cloud Playd (payant, rien à installer, 1
   mois d'essai gratuit sans carte — voir FYI plus bas). Si auto-hébergé : champ adresse + bouton
   "Tester la connexion" (→ `GET /health`, voir doc backend) + résultat coloré. CTA "Continuer".
   **Ce choix est celui de `feat_hosting`, pas de `feat_auth`** — le routeur de `shared` enchaîne
   `feat_hosting` (about → setup) → `feat_auth` (login), les deux features ne s'importent jamais
   entre elles (voir §2, règle de dépendance) ; go_router est le seul endroit qui connaît la
   séquence.
2. **Login** — email + mot de passe, lien "mot de passe oublié", lien "changer" (revient à l'écran
   1), lien vers register.
3. **Register** — email, username, mot de passe + confirmation (règle : 10 caractères minimum).
4. **Mot de passe oublié** — email, CTA "Envoyer" (`PasswordResetToken`, déjà en base côté backend).
5. **E-mail envoyé** — confirmation, boutons "Ouvrir le lien de réinitialisation" (simulateur, dans
   la vraie app ce sera un deep link), "Renvoyer", retour au login.
6. **Réinitialisation** — nouveau mot de passe + confirmation, écran atteint via le deep link de
   l'e-mail (à câbler avec go_router, `uni_links`/`app_links` ou l'intégration deep link de
   go_router — à trancher au scaffolding).

**Déconnexion** : bouton dans Réglages → Compte (`feat_auth`), ouvre une confirmation (titre "Se
déconnecter ?", compte affiché, action destructive rouge vs "Rester connecté") avant d'appeler
`POST /auth/logout` et de revenir à l'écran login.

**⚠️ TMDB — pas cohérent entre deux écrans du même handoff** : l'écran Réglages →
**Hébergement** (le menu principal) a maintenant un vrai encart pédagogique ("Elle se renseigne sur
le serveur, pas dans l'app", exemple `TMDB_API_KEY=votre_clé # .env du serveur`, lien externe vers
un futur `docs/SELF_HOSTING.md`) — **exactement la bonne explication**, aucun champ de saisie sur
cet écran. Mais l'écran Réglages → **Serveur** (une sous-page différente, atteinte depuis le même
menu) a **toujours** un champ `Clé API TMDB (côté serveur)` de type mot de passe où l'utilisateur
est invité à taper une valeur — la relabellisation ("côté serveur") et le nouveau hint ne
suppriment pas la contradiction : deux écrans de la même hiérarchie de réglages se contredisent
sur le fait qu'il existe, ou non, un champ à remplir. Les deux lectures possibles restent celles du
paragraphe "Écarts" ci-dessous, **avec maintenant une piste concrète pour l'option 1** (supprimer le
champ) : le lien "tutoriel d'auto-hébergement" de l'écran Hébergement pourrait tout simplement
remplacer le champ de l'écran Serveur.

**Documentation (README, tutoriels)** : **décision finale, revenue à ce que fait déjà le handoff**
— pas de rendu Markdown in-app (ça alourdirait le binaire pour un besoin ponctuel), chaque doc est
une carte `PlaydBanner` (déjà dans le design system) écrite en dur + un lien externe (`url_launcher`)
vers la page GitHub **rendue** (pas l'URL Markdown brute). Réutilisable pour le README, le futur
tutoriel clé TMDB, et le futur tutoriel self-hosting, sans nouveau composant `ui_kit` à concevoir.
Voir §2 (ligne Documentation) et §3 (`feat_hosting`).

**FYI, pas une décision d'archi** : Cloud Playd affiche maintenant un **essai gratuit d'un mois
sans carte** (`trialUsed`/`trialDaysLeft`) avant de passer à l'abonnement à 3 €/mois — détail
produit découvert dans le handoff, mentionné ici mais pas à figer, cohérent avec le fait que
toute la feature abonnement reste différée (voir plus bas).

### Écarts identifiés entre le handoff et le backend actuel (à trancher)

- **Changement de mot de passe** : le handoff a un écran dédié (Réglages → Compte → Modifier le
  mot de passe), mais ni `PATCH /users/me` ni aucune autre route backend ne l'accepte aujourd'hui
  (vérifié : `UpdateMeDto` n'a pas de champ mot de passe). **Backend à faire évoluer** avant de
  construire cet écran (nouvel endpoint, ou extension de `PATCH /users/me` avec vérification du
  mot de passe actuel) — pas un blocage pour le reste de l'app, juste cet écran précis.
- **Compteur de complétions d'une liste** (`runs`, badge "×N" quand une liste a été entièrement
  revue) : présent dans le prototype, absent du modèle `List` du backend. Pour rester cohérent
  (source de vérité serveur, pas de state local non synchronisé — voir §8), **ce compteur devrait
  être un champ backend** plutôt qu'être recalculé/stocké seulement côté app. À ajouter au schéma
  backend dans un futur incrément, pas requis pour le MVP de l'app.
- **Casting, bande-annonce, titres similaires** (fiche série/film) : le handoff les affiche, TMDB
  les fournit (`append_to_response=credits,videos,similar`), mais le module `catalog` du backend
  ne les expose pas aujourd'hui. Ces sous-sections de la fiche série/film sont **bloquées côté
  backend** tant que `catalog` n'est pas étendu — à prévoir comme tâche backend séparée, pas
  quelque chose que l'app peut contourner elle-même (l'app ne doit jamais appeler TMDB
  directement, voir §1/README backend).
- **Préférences d'affichage par titre** ("masquer les épisodes vus" pour une série, "masquer de la
  filmothèque" pour un film, via le menu Personnaliser ou Réglages → Bibliothèque séries/films) et
  **services de streaming préférés** (Réglages → Services d'abonnement, ordre d'affichage dans "Où
  regarder") : **décision revue** — la version précédente disait "restent locales à l'appareil,
  `shared_preferences`", mais du pur local sans sync signifie **tout perdre au changement de
  téléphone/réinstall** (remarque de l'utilisateur, correcte). Pour rester cohérent avec "le
  backend est la seule source de vérité, pas de mode local pur" (§8), ces préférences doivent
  **devenir des champs backend** comme le compteur de complétions de liste : `hideWatchedEpisodes`
  (booléen sur `UserShow`, ou son équivalent film une fois le statut film résolu ci-dessous),
  `preferredWatchProviders` (champ sur `User`, tableau d'identifiants de provider). Seules les
  préférences vraiment anodines à reconfigurer en quelques secondes (thème, langue, mode liste/
  grille, lecture auto — voir §2 `shared_preferences`) restent purement locales : la distinction
  n'est pas "c'est un réglage d'affichage" mais "ça a coûté un effort à configurer et ça ferait mal
  à perdre".
- **Statut favori/watchlist pour les films — gap découvert en creusant cette question** : le
  handoff prévoit "Favoris" et "Regarder plus tard" pour les films via le même menu `···` que les
  séries (voir §4, table, et la page "Films préférés"), mais **vérifié dans le schema** :
  `Movie` n'a que `watchEvents`/`ratings`, aucun équivalent de `UserShow` (pas de
  `isFavorite`/`isWatchlist`/`isArchived` pour un film). Contrairement aux items ci-dessus, ce
  n'est pas juste un champ à ajouter : ça manque une table entière côté backend. **Bloque
  réellement** la page "Films préférés" et l'entrée "Regarder plus tard" du menu `···` pour un
  film — à traiter avant de construire ces écrans précis, pas un blocage pour le reste de `tracking`.
- **Adresse du serveur éditable à l'exécution** : le handoff a un écran Réglages → Serveur qui
  permet de saisir/tester l'adresse du backend self-hosted **sans recompiler l'app** — c'est mieux
  que notre approche actuelle (uniquement `--dart-define` à la compilation). **À adopter** : garder
  `--dart-define` comme valeur par défaut (utile pour `template_app`), mais la rendre modifiable et
  persistée localement (`shared_preferences`) depuis les réglages — voir §2, ligne Config
  d'environnement, mise à jour en conséquence.
- **⚠️ Champ "Clé API TMDB" côté app — pas résolu, deux écrans du handoff se contredisent** :
  l'écran Réglages → **Hébergement** a un encart pédagogique complet (exemple `.env`, lien vers un
  tutoriel) et aucun champ de saisie — la bonne explication. Mais l'écran Réglages → **Serveur**,
  atteint depuis le même menu, a **toujours** un champ de saisie `Clé API TMDB` (`type: password`)
  qui contredit directement cet encart. Deux lectures possibles, **à trancher avec toi, pas supposé
  ici** :
  1. Le champ de l'écran Serveur est un reliquat à supprimer — l'app ne demande que l'adresse du
     serveur partout, la clé TMDB se configure exclusivement via `.env`/`docker-compose` côté
     déploiement (cohérent avec l'architecture backend actuelle, aucun changement requis). Le
     handoff donne une piste concrète pour cette option : remplacer le champ par le même lien vers
     le tutoriel d'auto-hébergement que sur l'écran Hébergement.
  2. Le champ doit rester, et l'app doit réellement pouvoir configurer la clé TMDB de son propre
     serveur à distance — ce qui suppose un **nouvel endpoint backend admin/config** (non authentifié
     par JWT utilisateur classique, plutôt une auth "propriétaire de l'instance") pour écrire
     `TMDB_API_KEY`, absent aujourd'hui et pas anodin à sécuriser (c'est un secret d'infrastructure,
     pas une donnée utilisateur).
  L'option 1 est la plus cohérente avec tout ce qui a été acté jusqu'ici (golden rule, backend
  source de vérité, pas de complexité inutile) — recommandation, pas une décision prise à ta place.
- **Documentation (README, tutoriels) — décision finale** : pas de rendu Markdown in-app ni de docs
  embarquées (ça alourdirait le binaire pour un besoin ponctuel, le texte des docs lui-même étant
  négligeable en taille). Revient à ce que fait déjà le handoff : cartes `PlaydBanner` écrites en
  dur (About, futurs tutoriels TMDB/self-hosting) + lien externe (`url_launcher`) vers la page
  GitHub **rendue** (pas l'URL Markdown brute `raw.githubusercontent.com`, illisible telle quelle).
  Voir §2 (ligne Documentation) et §3 (`feat_hosting`).
- **Notifications** : vérifié dans le prototype — contenu **entièrement personnel** ("nouvel
  épisode disponible", "House of the Dragon revient lundi", "sauvegarde exportée"), **aucun
  contenu social** (pas de like/commentaire/abonné). Légitimement dans le périmètre, mais pas
  d'équivalent backend aujourd'hui (`DeviceToken` existe pour l'enregistrement des tokens push,
  pas de notion de flux/historique de notifications). **Recommandation pour le v1** : rester
  minimal — notifications push natives (OS) déclenchées depuis les dates de diffusion du catalog
  pour les séries suivies, sans écran "historique" dédié ni stockage backend d'un flux — un
  historique synchronisé multi-appareils serait une vraie feature backend à part, pas nécessaire
  au MVP.

### Fonctionnalité prévue, différée : hébergement + abonnement (précisé après le §4)

Contrairement à ce que disait la version précédente de cette section, le mode "cloud géré" +
abonnement du handoff **n'est pas un non-objectif** : le projet publiera l'app sur les stores avec
un vrai choix pour l'utilisateur — self-hosting gratuit ou abonnement payant à une offre hébergée
gérée par le projet (voir §1, golden rule). Ça change ce qui était écrit au §8 (corrigé). Ce qui
reste vrai, en revanche : ce n'est **pas construit dans ce premier scaffolding**.

- **Ce qui est construit maintenant** : `feat_hosting` avec le choix de mode et l'écran self-hosted
  (adresse serveur, test de connexion) — voir §2/§3.
- **Ce qui est différé** (`AppFeatures.cloudSubscriptionEnabled = false` dans les deux apps pour
  l'instant) : instance/région gérée, plan d'abonnement, moyen de paiement, factures. Nécessite un
  **vrai backend multi-tenant** (provisioning d'instance par utilisateur payant) et une intégration
  de paiement (probablement in-app purchase App Store/Play Store plutôt qu'un système de facturation
  web propre, à trancher le moment venu) — un chantier backend à part entière, largement plus gros
  que les 4 items déjà listés dans
  [`another_tvtime_backend/docs/flutter-handoff-backend-changes.md`](../another_tvtime_backend/docs/flutter-handoff-backend-changes.md),
  volontairement pas détaillé ici tant qu'il n'est pas planifié.
- **Concerne les deux apps** : `template_app` garde le même package `feat_hosting`, un forkeur
  pourrait un jour proposer sa propre offre hébergée payante via le même mécanisme — ce n'est pas
  une exclusivité de `playd`.

### Éléments du handoff à écarter du périmètre (confirmés hors scope, pas une régression)

- **"Garder les données sur cet appareil" / `privateProfile`** (Réglages → Vie privée) : ce toggle
  du prototype implique un mode où rien ne synchronise avec le serveur tant qu'on n'exporte pas
  manuellement — ça **contredit** le non-objectif déjà acté "pas de mode offline complet en v1,
  l'app suppose une connexion à l'instance backend" (§8). **Non adopté pour le v1** : le backend
  reste la seule source de vérité, pas de mode local pur en parallèle. Noté ici plutôt que
  silencieusement ignoré, pour que ce soit un choix explicite et pas un oubli.
- **Sondage communautaire et courbe de notes communautaires** (fiche série, onglet "À propos") :
  agrégats inter-utilisateurs que le backend ne calcule jamais et ne calculera pas sans revisiter
  la décision "pas de social" du projet (voir `docs/tvtime-gdpr-export-reference.md` côté backend).
  **À retirer** de l'implémentation ; la note TMDB globale (`vote_average`, déjà disponible via
  `catalog`) peut rester affichée seule, ce n'est pas un agrégat propre à nos utilisateurs.
- **"Série/Film ajouté par N personnes"** (résultats de recherche, onglet Explorer) : même raison
  — compteur d'usage agrégé entre utilisateurs, jamais exposé par notre backend. **À retirer** ou
  remplacer par une métadonnée neutre (année, type) dans les résultats de recherche.

## 5. Détail : statut de suivi d'une série

Le backend modélise le statut d'une série suivie avec **quatre booléens indépendants** sur
`UserShow` (`isFollowing`, `isFavorite`, `isWatchlist`, `isArchived` — voir
`another_tvtime_backend/prisma/schema.prisma`), pas un enum unique. Le menu contextuel du handoff
("Favoris", "Regarder plus tard", "Arrêter de regarder / Reprendre", "Supprimer") correspond à
`isFavorite`/`isWatchlist`/`isArchived`/`DELETE /tracking/shows/:tmdbId` (voir §4). "Partager" est
purement client (copie presse-papiers, pas d'appel backend). "Personnaliser" (notifications,
masquer les épisodes vus) n'a pas d'équivalent backend, mais **n'est pas hors périmètre** : c'est
une préférence d'affichage locale résolue au §4/§2 (`shared_preferences`, non synchronisée pour le
v1) — mise à jour depuis la version précédente de cette section, qui la classait par erreur comme
hors périmètre faute d'avoir vu concrètement ce qu'elle fait. Le modèle Dart `ShowTrackingStatus`
doit rester quatre booléens, pas un enum, pour matcher exactement le DTO
`PATCH /tracking/shows/:tmdbId`.

## 6. Thème

**Source de vérité : `handoff/README.md`**, explicitement écrit comme référence consolidée pour
`ui_kit` ("Objet : source de vérité visuelle de Playd"). Fidélité "high-fidelity" — repris au
pixel/à la milliseconde près via `ui_kit` (**sauf la nav bar**, voir plus bas). Ne pas dupliquer
l'intégralité des tableaux de tokens ici — le handoff en a déjà une version complète, prête à
l'emploi ; ce qui suit n'est qu'un résumé structurant.

- **Thème double, dark + light, tous deux de premier plan** — correction d'une version précédente
  de cette section qui disait "pas de mode clair, non prioritaire" : le handoff donne les **deux
  jeux de valeurs pour les mêmes tokens** (dark canonique, light dérivé), pas un Material 3
  générique en roue de secours. `PlaydTheme.dark` / `PlaydTheme.light`, choix system/manuel via
  `ThemeMode`.
- **Tokens couleur, par catégorie** (noms exacts dans `PlaydColors extends ThemeExtension`, jamais
  de couleur en dur dans un widget) : `surface.*` (base/raised/sunken/inset/media), `border.*`
  (subtle/strong/track), `content.*` (primary/secondary/tertiary/muted), `brand.*` (accent
  `#efbe4e` inchangé, accentPressed, accentText, accentSoft, onAccent, logoRamp 1-5, iconCanvas,
  splashTile/Crest), `state.*` (success, **`fullCompleted` — nouveau, violet `#9d6bff`/`#6d3fd9`,
  pour une œuvre entièrement terminée : tout vu ET plus de saison à venir, distinct de "vu mais
  encore en production"** ; danger, dangerSoft, info, online), `action.markSurface`,
  `overlay.scrim`.
- **Typographie** : toujours Archivo seule (voir §2, embarquée en asset), mais échelle nommée et
  complète maintenant (`display.lg/md`, `title.xl/lg/md/sm`, `subtitle`, `body`, `bodySm`,
  `caption`, `label`, `labelSm`, `chip`, `button`/`buttonSm`, `navLabel`, `mono`) — à porter en
  `PlaydText.*`, pas en styles ad hoc par écran.
- **Tokens de forme** : `space` (4/8/12/16/24/34), `radius` (0 par défaut, 3 = quasiment partout —
  cartes, champs, modales, encarts ; 6 = vignette de grille ; 10 = carte de stat ; 999 = pilule),
  `elevation` (`e1`/`e2` + le liseré de verre `glass.edge`), `motion` (durées/courbes nommées :
  `fast`/`base`/`glass`/`spring`). Cible tactile ≥ 44 px partout.
- **Composants `ui_kit`** : 19 widgets `Playd*` listés au complet dans le design system (boutons,
  chip/badge, champ de texte, switch/radio/pastille "vu", onglets de verre, en-tête de section,
  rangée de réglage, ligne média, vignette d'affiche, barre de progression, carte stat, carte de
  liste, bannière, dialog/sheet, toast, empty state, marque, avatar) — voir §3 pour l'arborescence
  de fichiers exacte, calquée sur celle proposée par le handoff lui-même.
- **Icônes** : Lucide (voir §2).
- **Marque (`PlaydLogoMark`)** : un « P » de 5 carrés jaunes (grille 2×3, cellule bas-droite vide),
  rampe `brand.logoRamp.1…5` (`#ffd60f` → `#e8a30a`), **toujours posé sur son carré
  `brand.iconCanvas` (`#2a2a2a`), jamais à nu**. Tailles de cellule selon contexte : 17/9/7/6 px
  (grand logo/auth/en-tête/badges). Distinct de l'accent `#efbe4e`, qui ne change pas.
- **Icône d'app** : fond `#2a2a2a` (`brand.iconCanvas`) avec une texture très subtile (dégradés
  conic/radial à faible opacité, quasiment plate), P centré — cohérent avec `PlaydLogoMark`. À
  générer en PNG (1024/512/192/180/120/64/48) au scaffolding, voir `handoff/Playd Icon.dc.html`.
- **Splash screen** : inchangé (plein écran `#070707`/`#090909`, `PlaydLogoMark` fixe au centre
  cellule 42 px, animation "onde" aux formules fournies, 2600 ms ou tap pour passer). Voir §2/§3.
- **Barre de navigation basse — "liquid glass", pas pleine largeur** : `handoff/README.md` sert de
  **maquette de référence visuelle**, pas de spec littérale à recopier au pixel — décision
  explicite de l'utilisateur : le rendu final vient d'**`adaptive_platform_ui`** (voir §2), pas
  d'un composant custom peint à la main, pour avoir le meilleur rendu natif et la meilleure
  couverture sur l'ensemble des OS mobiles ciblés plutôt qu'un seul rendu identique partout. Le
  handoff confirme explicitement que **cette barre n'est pas dans `ui_kit`** ("fournie par une
  librairie externe") — elle vit dans `shared/src/navigation/` (voir §3). Ce que la maquette donne
  à viser lors de la configuration du composant :
  - Capsule flottante centrée, largeur au contenu (pas pleine largeur).
  - Teinte or `#efbe4e`/`rgba(239,190,78,.20)` pour l'état sélectionné, 4 onglets (icônes Lucide
    `tv`/`clapperboard`/`search`/`user`).
  - Comportement de compression au scroll (labels masqués, capsule réduite) — **à vérifier au
    scaffolding** si `adaptive_platform_ui` l'expose nativement (le package mentionne "minimize
    behavior" pour son UITabBar) ou si ça reste à composer par-dessus.
  - Sur Android / iOS < 26, le rendu du package (Material 3 / Cupertino) ne reproduira pas
    littéralement l'effet verre de la maquette — **compromis assumé**, pas un défaut à corriger.
  - Les barres de sous-onglets sticky (À voir/À venir), elles, **restent dans `ui_kit`**
    (`PlaydGlassTabs` → `ui_kit/src/components/playd_glass_tabs.dart`, voir §3) — le design system
    les traite comme un composant interne, contrairement à la nav bar principale.

## 7. Conventions

Les conventions ci-dessous ont vocation à devenir, plus tard, un skill dédié pour faciliter le
développement du projet (rappelé par l'utilisateur) — les garder concrètes et vérifiables plutôt
que de simples intentions est donc d'autant plus important.

- **Lint : `very_good_analysis`**, pas `flutter_lints` — tranché : pairing naturel avec
  `very_good_cli` déjà retenu pour le scaffolding (même éditeur, ruleset plus strict que le défaut
  officiel), plutôt que de mélanger deux écosystèmes d'outillage différents sans raison.
- **Conventions de nommage Bloc/Cubit** (officielles, [bloclibrary.dev](https://bloclibrary.dev/naming-conventions/)) :
  - Events au **passé** (`LoginSubmitted`, pas `SubmitLogin`) — un event représente quelque chose
    qui s'est déjà produit du point de vue du Bloc/Cubit.
  - States comme des **noms** (photographie à un instant T), pas des verbes.
  - **Un seul style de state pour tout le projet**, pas un mélange au cas par cas : soit des
    sous-classes (`AuthInitial`/`AuthSuccess`/`AuthFailure`), soit une classe unique + un enum de
    statut (`AuthState { status: AuthStatus.initial/success/failure, ... }`) — à choisir au
    scaffolding et documenter ici une fois choisi.
  - **`sealed class`** (Dart 3, disponible en 3.13) pour les events et les states de chaque
    Bloc/Cubit — permet un `switch` exhaustif vérifié à la compilation plutôt que des `if is` en
    cascade ou un `default` qui masque un cas oublié.
- **Application mécanique du graphe de dépendance entre packages** (voir §2, "Règles de
  dépendance") : via les contraintes de `pubspec.yaml` dans un Dart/pub workspace — transforme la
  règle "un `feat_xxx` ne doit pas importer un autre `feat_xxx`" en erreur de compilation plutôt
  qu'en convention vérifiée seulement au lint/CI. À mettre en place dès le scaffolding, pas ajouté
  après coup une fois que des violations existent déjà.
- **Pattern "Module + callbacks" pour la coordination inter-features — à évaluer au scaffolding,
  pas encore tranché** : alternative/complément à "le routeur de `shared` connaît le nom de toutes
  les routes" (voir §2/§3, Setup → Login par ex.) — chaque feature exposerait un widget d'entrée
  recevant des callbacks typés pour la navigation, sans jamais connaître le nom des routes des
  autres features. Séduisant pour le découplage, mais à confronter concrètement à notre organisation
  (`shared` connaît déjà toutes les features par construction, donc le gain est moins évident que
  dans une architecture qui viserait du lazy-loading par feature) avant de l'adopter.
- **Barrel file par écran**, pas seulement par package : en plus du barrel `feat_xxx.dart` déjà
  prévu par package (voir §3), un barrel par écran dans `presentation/` (ex.
  `presentation/login/login.dart` qui réexporte la vue + son Cubit) — prépare un éventuel
  lazy-loading ciblé d'un écran précis si l'app grossit, sans coût aujourd'hui.
- **Doc comments (`///`) obligatoires sur tout le `domain/` public** : interfaces `Repository`,
  modèles/entités. Objectif précis : quelqu'un doit pouvoir comprendre le contrat d'une feature en
  lisant seulement son `domain/`, sans ouvrir `data/`/`presentation/` — cohérent avec `domain/` qui
  est déjà, par construction, la seule couche que toutes les autres dépendent (voir §2).
- **Limite de taille/imbrication sur `build()`** : extraire en sous-widget au-delà d'environ 100
  lignes ou 3-4 niveaux d'imbrication — encourage la composition (widgets nommés, testables
  isolément) plutôt qu'un `build()` monolithique difficile à relire.
- Tests : unitaires sur les `Repository` (mapping JSON ↔ modèle, gestion d'erreurs — mocks via
  **`mockito`**, ex. mock d'`AuthApi`/`api_client` pour tester `AuthRepositoryImpl` sans réseau) et
  les Cubits/Blocs métier (**`bloc_test`**, `blocTest<Cubit, State>(...)` pour asserter les
  séquences d'états sans boilerplate) ; pas de golden tests dans le périmètre v1 (coût d'entretien
  élevé pour un projet solo/communautaire).
- **Miroir `test/` ↔ `lib/`** : tout fichier créé dans `lib/src/...` a son équivalent
  `test/src/..._test.dart` à la même profondeur de dossiers (ex. `lib/src/data/auth_repository.dart`
  → `test/src/data/auth_repository_test.dart`), dans chaque package du monorepo (`feat_xxx`,
  `shared`, `ui_kit`, `api_client`). Une divergence entre les deux arborescences est un signal
  qu'un fichier n'est pas testé — pas une règle qui impose 100 % de couverture, mais qui rend un
  trou visible d'un coup d'œil plutôt que caché dans un `test/` qui a dérivé de `lib/` avec le temps.
- Nommage fichiers : `snake_case.dart`, un fichier = une classe publique principale.
- i18n : `fr` + `en` a minima dès le départ (jamais de texte en dur), un fichier de traduction par
  package `feat_xxx` (voir §2, ligne Localisation, et §3). **Langue par défaut = celle du téléphone**
  (`Locale` système, via `flutter_localizations`/`MaterialApp.supportedLocales`) ; si la langue du
  téléphone n'a pas de traduction dans un `feat_xxx` donné, repli sur l'anglais pour cette feature
  (pas le français) — cohérent avec l'esprit "self-hostable par n'importe qui" : l'anglais est le
  dénominateur commun international, pas une préférence de l'auteur du projet.

## 8. Non-objectifs explicites

- Pas de social : ni commentaires, ni amis, ni réactions, ni notifications d'autres utilisateurs,
  ni badges/leaderboard, **ni agrégats inter-utilisateurs** (sondage communautaire, courbe de
  notes communautaires, "ajouté par N personnes" — présents dans le handoff visuel mais à retirer,
  voir §4) — le backend ne les expose pas et ne les calculera pas sans revisiter cette décision.
- Pas de SDK de plateforme tiers par défaut (pas de Firebase Auth/Analytics/Crashlytics, pas
  d'Amplitude) — cohérent avec la golden rule coûts/pérennité. Du monitoring optionnel et
  auto-hébergeable (ex: Sentry self-hosted) pourra être ajouté plus tard, jamais comme dépendance
  dure.
- Pas de mode offline complet en v1 (pas de base locale genre Drift/Isar) — l'app suppose une
  connexion à l'instance backend. Un cache léger (`cached_network_image` pour les images, cache
  HTTP dio pour les réponses catalog) suffit pour l'expérience v1. Le toggle "garder les données
  sur cet appareil" vu dans le handoff (§4) n'est **pas** adopté pour cette raison — le backend
  reste la seule source de vérité, pas de mode local pur en parallèle.
- **Corrigé** (l'affirmation précédente ici était fausse, voir §1/§4) : le paiement in-app **fait**
  partie de la même codebase, ce n'est pas un déploiement/flavor à part — l'app publiée sur les
  stores proposera un choix self-hosted gratuit / abonnement payant à une offre hébergée, dans les
  deux apps (`playd` et `template_app`), via `feat_hosting`. Le non-objectif réel : **ne pas
  le construire dans ce premier scaffolding** — `AppFeatures.cloudSubscriptionEnabled` reste faux
  jusqu'à ce que le backend multi-tenant + l'intégration de paiement soient un chantier planifié
  (voir §4, "Fonctionnalité prévue, différée").

## Prochaines étapes

1. Tu relis/amendes ce document directement (texte libre) jusqu'à ce qu'il te convienne —
   notamment §3 (structure à réécrire pour le monorepo Melos) et §7 (lint strict ou non).
2. Une fois validé, on scaffolde le projet (`flutter create`, arborescence §3, packages §2) et on
   génère les modèles Dart à partir du contrat API exact du backend (routes + DTOs détaillés,
   au-delà du résumé du §4).
3. Au scaffolding, **valider concrètement le tree-shaking des `feat_xxx` désactivées via
   `AppFeatures`** (§2, ligne Feature flags par app) — build `template_app` avec une feature
   désactivée, comparer la taille du binaire (`flutter build apk --analyze-size` ou équivalent
   iOS) avec/sans cette feature référencée quelque part dans le code de l'app. Si le gain n'est
   pas réel, documenter que le flag reste utile pour masquer l'UI/DI mais ne réduit pas la taille
   du binaire — ne pas le présenter comme un gain acquis sans l'avoir mesuré.
4. **Côté backend**, items identifiés en fusionnant le handoff (§4) listés séparément dans
   [`another_tvtime_backend/docs/flutter-handoff-backend-changes.md`](../another_tvtime_backend/docs/flutter-handoff-backend-changes.md)
   (endpoint de changement de mot de passe, compteur de complétions sur `List`, extension
   `catalog` casting/bande-annonce/similaires, endpoint `/health`, `hideWatchedEpisodes` sur
   `UserShow`, `preferredWatchProviders` sur `User`, et — plus gros — un modèle de statut
   favori/watchlist pour les films, qui n'existe pas du tout aujourd'hui). Aucun n'est bloquant
   pour démarrer le scaffolding Flutter — ils bloquent seulement les écrans/sous-sections
   concernés (le dernier bloque spécifiquement "Films préférés" et une partie du menu `···` film).
5. Aucune commande Flutter ni fichier de code n'est créé avant cette validation.
