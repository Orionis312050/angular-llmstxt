# angular-llmstxt

Version **offline et auto-contenue** du [llms.txt d'Angular](https://angular.dev/llms.txt), pensée pour les environnements fermés (knowledge base d'IA ne pouvant pas fetcher de liens externes).

## Contenu

- `llms.txt` : reprend la structure exacte du llms.txt officiel d'angular.dev. Chaque entrée pointe vers un fichier local de `docs/` contenant la **section inline complète** de la page (titre + URL source + texte intégral) — aucun fetch externe requis.
- `docs/part01.md` … `docs/part09.md` (~1 Mo au total) : les 88 pages inlinées (guides components, templates, directives, signaux, DI, RxJS, HTTP, formulaires, routing, SSR, tests, animations, i18n, zoneless...), la référence CLI commande par commande, le guide de style, la sécurité, CONTRIBUTING, etc.
- La référence API (`/api`, index de plusieurs milliers de symboles généré au build) est documentée avec le pattern d'URL des symboles individuels.

## Source

Contenu issu du dépôt officiel [`angular/angular`](https://github.com/angular/angular) (`adev/src/content`, branche `main`), résolu via la table de navigation `adev/src/app/routing/navigation-entries/index.ts`.

## Utilisation

Pointez votre outil/knowledge base sur le dépôt ou les fichiers raw, par exemple :

```
https://raw.githubusercontent.com/Orionis312050/angular-llmstxt/main/llms.txt
https://raw.githubusercontent.com/Orionis312050/angular-llmstxt/main/docs/part01.md
```
