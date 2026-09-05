# Playd — Design System (export pour Claude Code)

## Objet
Source de vérité visuelle de Playd (tracker de séries/films, thème sombre canonique + clair dérivé),
destinée à devenir un **package melos `ui_kit` Flutter** : tokens en `ThemeExtension` + widgets.
Le fichier `Playd Design System.dc.html` est la **planche de référence** (tokens et specimens en
sombre ET clair, côte à côte). `TVTime Clone.dc.html` est l'app complète qui consomme ce système —
à lire pour voir chaque composant en contexte. Ce sont des **références de design en HTML**, pas du
code à porter : reconstruire en Flutter avec les patterns du projet.

Police unique : **Archivo** (400/500/600/700/800). Rayon par défaut **0**, rayon de composant **3 px**,
pilules **999 px**. Toute cible tactile **≥ 44 px**.

---

## 1. Tokens couleur — `playd_colors.dart` / `PlaydColors extends ThemeExtension`

| Token | Sombre | Clair | Usage |
| --- | --- | --- | --- |
| `surface.base` | `#000000` | `#ffffff` | Fond d'écran, scroller |
| `surface.raised` | `#141414` | `#f5f4f2` | Cartes, encarts, hover de rangée |
| `surface.sunken` | `#1c1c1c` | `#ebe9e6` | Blocs de listes (pages de tracking) |
| `surface.inset` | `#0f0f0f` | `#f0eeeb` | Champs, options non choisies |
| `surface.media` | `#171717` | `#e4e1dc` | Réserve d'affiche / vignette vide |
| `border.subtle` | `#2a2a2a` | `#e2dfda` | Filets 1 px |
| `border.strong` | `#444141` | `#c9c5be` | Bordure de champ, bouton secondaire |
| `border.track` | `#3a3a3a` | `#d5d1ca` | Rail de progression, pointillé |
| `content.primary` | `#ffffff` | `#141312` | Titres, valeurs |
| `content.secondary` | `#e8e6e6` | `#3d3a35` | Sous-titres, corps |
| `content.tertiary` | `#9b9797` | `#6b6660` | Méta, description, hint |
| `content.muted` | `#7d7979` | `#8c8781` | Onglet inactif, désactivé, chip |
| `brand.accent` | `#efbe4e` | `#efbe4e` | Action primaire, sélection, progression en cours |
| `brand.accentPressed` | `#d9a53a` | `#d9a53a` | Hover / pressed |
| `brand.accentText` | `#f6d689` | `#8f6415` | Texte or lisible (lien, onglet actif) |
| `brand.accentSoft` | `#1c1810` | `#fdf3dc` | Fond d'option sélectionnée |
| `brand.onAccent` | `#201e1d` | `#201e1d` | Encre sur fond or |
| `brand.logoRamp.1…5` | `#ffd60f` `#f8c608` `#f4b909` `#eeae08` `#e8a30a` | idem | Les 5 carrés du P |
| `brand.iconCanvas` | `#2a2a2a` | `#2a2a2a` | Canevas de l'icône (et fond du P dans le lockup) |
| `brand.splashTile` / `Crest` | `#070707` / `#404040` | idem | Carreau du splash / crête de l'onde |
| `state.success` | `#76d919` | `#4e8f0d` | Vu, essai gratuit, test OK |
| `state.fullCompleted` | `#9d6bff` | `#6d3fd9` | Œuvre achevée : tout vu **et** plus de saison à venir |
| `state.danger` | `#e5533c` | `#c23a24` | Erreur, favori (cœur), suivi arrêté |
| `state.dangerSoft` | `#1a0e0b` | `#fbe9e4` | Fond du bandeau d'erreur |
| `state.info` | `#3b8cff` | `#1f6fe0` | Valeur cliquable d'une rangée de réglage |
| `state.online` | `#5ecb7e` | `#3f9a5b` | Pastille de sync active |
| `action.markSurface` | `#f3f2f2` | `#ffffff` | Cercle « marquer vu » non vu (icône `#605d5d`) |
| `overlay.scrim` | `#000` 72 % | `#000` 45 % | Voile de modale / feuille |

Le sombre est la référence ; le clair n'est qu'un second jeu de valeurs pour les **mêmes noms**.
Aucun widget ne code une couleur en dur : tout passe par
`Theme.of(context).extension<PlaydColors>()!`.

## 2. Typographie — `playd_typography.dart` / `PlaydText`

| Token | Style | Usage |
| --- | --- | --- |
| `display.lg` | 800 · 34/1.04 · -.02em | Titre d'accueil / auth |
| `display.md` | 800 · 32/1.06 · -.02em | Titre README de premier lancement |
| `title.xl` | 700 · 30/1.05 | Titre de fiche |
| `title.lg` | 700 · 25/1 | En-tête de section (profil) |
| `title.md` | 700 · 19/1 | Code d'épisode, valeur chiffrée |
| `title.sm` | 600 · 17/1.2 | Titre de film, libellé de rangée |
| `subtitle` | 400 · 17/1.2 | Sous-titre de fiche, valeur de réglage |
| `body` | 400 · 15/1.45 | Paragraphe, corps d'encart |
| `bodySm` | 400 · 14/1.25 | Titre d'épisode, description de puce |
| `caption` | 400 · 12.5/1.4 | Hint de champ, note |
| `label` | 600 · 12/1 · .1em · caps | Étiquette de champ, titre de groupe |
| `labelSm` (kicker) | 700 · 11/1 · .14em · caps | Kicker d'encart |
| `chip` | 700 · 12/1 · .09em · caps | Chip de groupe, badge, onglet |
| `button` / `buttonSm` | 700 · 15 ou 13/1 · .08em · caps | Libellé de pilule |
| `navLabel` | 600 · 10.5/1 · .02em | Libellé de tab bar |
| `mono` | 600 · 12.5/1.4 monospace | Config, adresse de serveur |

## 3. Tokens de mise en forme — `playd_tokens.dart`

- **space** : `xs 4` · `sm 8` · `md 12` · `lg 16` (gouttière écran) · `xl 24` · `2xl 34`
- **radius** : `none 0` (défaut) · `xs 3` (cartes, champs, encarts, modales) · `sm 6` (vignette de grille) · `md 10` (carte de stat) · `pill 999` (boutons, chips, badges)
- **elevation** : `e1 0 4 12 rgba(0,0,0,.28)` · `e2 0 14 34 rgba(0,0,0,.55)` · `glass.edge` = liseré interne `inset 0 .8 0 rgba(255,255,255,.34)` + `inset 0 0 0 .5 rgba(255,255,255,.14)`
- **motion** : `fast 250 ease` · `base 300 ease` · `glass 420 cubic-bezier(.22,1,.36,1)` · `spring 520 cubic-bezier(.34,1.42,.42,1)`

## 4. Composants — `ui_kit/lib/src/components`

| Widget | Variantes / props | Spécificités |
| --- | --- | --- |
| `PlaydButton` | `primary` · `secondary` · `ghost` · `danger` · `disabled` · `icon` | Pilule 999, libellé caps .08em, hauteur ≥ 44 (padding 16/18 avec label 15, 15/18 avec label 13). Primary = accent plein sur `onAccent`, hover `accentPressed`. Secondary = bordure 2 px `content.primary`, hover inversé. Ghost = texte `accentText`. Disabled = opacité 45 %. |
| `PlaydChip` / `PlaydBadge` | `group` · `outline` · `accent` · `premiere` · `count` | **Tout est entièrement arrondi** : pilule 999 pour les chips et badges, cercle parfait pour le badge compteur. Chip de groupe = fond `content.muted`, texte blanc. |
| `PlaydTextField` | `label` · `value` · `hint` · `obscure` · `focused` · `error` | Étiquette caps au-dessus, champ `surface.inset` + bordure `border.strong`, **rayon 3**, focus = bordure accent. Erreur = bandeau filet gauche 3 px `danger` sur `dangerSoft`, rayon 3. |
| `PlaydSwitch` · `PlaydRadio` · `PlaydWatchDot` | `on/off` · `selected` · `watched/unwatched` | Switch 50×30 pilule. **WatchDot** : 42 px de diamètre, non vu = `action.markSurface` + tick `#605d5d`, **vu = `state.success` + tick blanc**. |
| `PlaydGlassTabs` | `items` · `selectedIndex` | Sous-onglets sticky en verre : `rgba(6,6,6,.62)` + blur 22 sat 180, liseré bas 0.5 px, indicateur 3 px sous l'onglet actif. Clair : `rgba(255,255,255,.66)` + liseré `rgba(0,0,0,.10)`. |
| `PlaydSectionHeader` | `title` · `onTap` | Titre `title.lg` + chevron 22, sans filet. |
| `PlaydSettingRow` | `label/value` · `toggle` · `danger` | Rangée 56–64, filet 1 px `border.subtle`, valeur cliquable en `state.info` + chevron `content.muted`. |
| `PlaydMediaRow` | `show` · `episode` · `movie`, trailing `markWatched` | Affiche 100 px (74 dans la planche), colonne texte (chip de titre, `title.md`, `bodySm`), action 66 px avec `PlaydWatchDot`. Fond `surface.base` sur `surface.sunken`, rayon 3. |
| `PlaydPosterTile` | ratio 2/3, rayon 6, `progress` | **Aucun badge d'angle** : l'état se lit sur le rail de 6 px en pied (`border.track` + remplissage). En cours = `brand.accent` ; tout vu mais série encore en production = `state.success` ; tout vu et œuvre achevée (film vu, ou série finie) = `state.fullCompleted` ; suivi arrêté / archivé = `state.danger`. Titre 800/11 caps avec ombre portée. |
| `PlaydProgressBar` | `value` · `tone` | 4 px (carte) / 6 px (vignette) / 8 px (hero de fiche). Mêmes tonalités que ci-dessus. |
| `PlaydStatCard` | `label` · `parts[]` | Rayon 10, bordure `border.strong`, en-tête à filet + icône 16. |
| `PlaydListCard` | `name` · `why` · `count` · `progress` | 152×146, `surface.raised`, bordure `border.subtle`, hover bordure accent, rail 4 px. |
| `PlaydBanner` | `tone: accent · success · danger · info` | Filet gauche 3 px, fond `surface.raised`, **rayon 3**, kicker + titre + corps + CTA optionnel ; bloc mono de config également en rayon 3. |
| `PlaydDialog` / `PlaydSheet` | `title` · `body` · `actions` | Sur `overlay.scrim`, conteneur `#161616` (clair : `surface.base`) avec filet supérieur 3 px de la tonalité, **rayon 3**. |
| `PlaydToast` | `message` | Pilule 999 centrée, `action.markSurface` sur `onAccent`, ~2 s. |
| `PlaydEmptyState` | `message` · `action` | Bordure pointillée 1 px `border.track`, **rayon 3**, texte `content.muted`. |
| `PlaydLogoMark` | `size` · `withWordmark` | Le P jaune se pose **toujours** sur son carré `brand.iconCanvas` (#2a2a2a), comme l'icône d'app — jamais à nu. Grille 2 × 3, 5 cellules dans l'ordre de `brand.logoRamp`, le carré vaut 1/5 du canevas. Identique en clair et en sombre. |
| `PlaydAvatar` | 52 · 38 | Cercle bordé 2 px `content.primary`. |

La barre de navigation basse n'est **pas** dans le kit : elle sera fournie par une librairie
externe (nav bar « liquid glass »). Les sous-onglets en verre (`PlaydGlassTabs`) restent internes.

## 5. Arborescence visée

```
packages/ui_kit/
  lib/
    ui_kit.dart
    src/
      theme/
        playd_colors.dart      → tokens couleur (dark/light)
        playd_typography.dart  → PlaydText.*
        playd_tokens.dart      → space / radius / elevation / motion
        playd_theme.dart       → PlaydTheme.dark · PlaydTheme.light
      components/
        playd_button.dart          playd_chip.dart
        playd_text_field.dart      playd_switch.dart
        playd_glass_tabs.dart      playd_setting_row.dart
        playd_section_header.dart  playd_media_row.dart
        playd_poster_tile.dart     playd_progress_bar.dart
        playd_stat_card.dart       playd_list_card.dart
        playd_banner.dart          playd_dialog.dart
        playd_toast.dart           playd_empty_state.dart
        playd_logo_mark.dart
```

## 6. Règles de portage

- Verre : `BackdropFilter(ImageFilter.blur(sigmaX: 11, sigmaY: 11))` dans un `ClipRRect` +
  **couche de teinte en dégradé** — sur fond noir pur, le flou seul ne produit rien.
  Repli opaque (`surface.raised` à 92 %) si la transparence est désactivée.
- Jamais plus de deux niveaux de surface empilés : `base → sunken → raised`.
- Les images (affiches, vignettes, bannières) sont des réserves : elles viendront de **TMDB**,
  la clé d'API étant configurée **côté serveur** en auto-hébergé.
- Contraste : le texte or n'est jamais utilisé en corps sur fond clair — utiliser `brand.accentText`
  (`#8f6415` en clair).

## Fichiers du bundle

| Fichier | Contenu |
| --- | --- |
| `Playd Design System.dc.html` | La planche : tokens + composants en sombre et clair |
| `TVTime Clone.dc.html` | L'app complète consommant le système (contexte d'usage) |
| `Playd Icon.dc.html` | Propositions de logo, variante A retenue |
| `support.js`, `image-slot.js` | Runtime des prototypes (référence, à ne pas porter) |
| `_ds/modernist-…/styles.css` | Tokens du cadre de présentation de la planche |
