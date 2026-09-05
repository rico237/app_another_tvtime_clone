# Playd - R.I.P. TVTime

*This project is mainly vibe coded using Claude AI (Back & Front) - few manual changes are made*

*Lisez-moi · github.com/playd-app/playd*

## Retrouver TV Time, et le garder cette fois

J'ai suivi mes séries sur TV Time pendant treize ans. Quand l'app a fermé, je n'ai pas seulement
perdu un historique : j'ai perdu une habitude, une façon de retrouver mes séries chaque soir. Playd
est ma tentative de remettre exactement cette expérience en place — et de faire en sorte qu'elle ne
puisse plus disparaître du jour au lendemain, parce que le serveur est open source et que n'importe
qui peut l'héberger.

### L'auto-hébergement est le but, pas le repli

Une image Docker, une commande, et vous avez l'application entière : suivi des épisodes, listes,
statistiques, export. Aucune fonction n'est réservée aux payants, aucun compte n'est requis. C'est
le mode que je recommande — c'est celui qui rend l'app indépendante de moi.

### Vos données restent les vôtres

Tout vit en local ou sur votre serveur, et s'exporte en JSON à tout moment. Pas de publicité, pas
de traqueur, pas de profil de visionnage revendu. Si le projet s'arrête, votre instance continue de
tourner.

### L'abonnement paie mon serveur, rien d'autre

Si vous ne voulez pas administrer de machine, vous pouvez utiliser mon instance : elle coûte de
l'argent à maintenir et à garder en ligne, donc elle est payante — 3 €/mois, après un mois d'essai
sans carte. Ce n'est pas un péage sur des fonctionnalités, c'est le prix de l'hébergement. Vous
pouvez basculer vers votre propre serveur quand vous voulez, avec vos données.

---

## Pour aller plus loin

- [`ARCHITECTURE.md`](ARCHITECTURE.md) — structure du projet, stack technique, packages (document
  de conception de l'app Flutter, à ce stade pas encore implémentée).
- [`another_tvtime_backend/`](../another_tvtime_backend/) — l'API self-hostable que cette app
  consomme (NestJS + Postgres + TMDB), déjà fonctionnelle.
