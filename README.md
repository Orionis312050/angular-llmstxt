# angular-llmstxt

Édition **offline** de la doc Angular, à partir de https://angular.dev/llms.txt (+ contenu complet de chaque page inline).

## Structure

- `llms.txt` : index de toutes les pages (liens relatifs vers `docs/`).
- `docs/<categorie>/<feature>.md` : **une feature par fichier**, contenu complet inline.
- Chaque dossier de catégorie contient un `_category_.json` (compatible Docusaurus).
- Tous les cross-references sont des liens relatifs intra-dépôt : aucun fetch externe requis.
- `script/compilers.sh` : recompile `docs/` + l'index depuis le llms.txt complet offline.
- `script/llms.txt.part-*` : le llms.txt **complet** (88 pages inline), poussé en parties ; `script/compilers.sh` le réassemble automatiquement en `script/llms.txt`.

## Utilisation (environnement fermé)

```bash
git clone <ce dépôt> && cd angular-llmstxt
bash script/compilers.sh            # réassemble script/llms.txt puis compile docs/ + index llms.txt
bash script/compilers.sh --reassemble   # réassemble uniquement script/llms.txt
bash script/compilers.sh --from-docs    # reconstruit script/llms.txt depuis docs/
```

## Source

Généré depuis `angular/angular` (`adev/src/content`, mapping via `adev/src/app/routing/navigation-entries/index.ts`), à partir de https://angular.dev/llms.txt.
