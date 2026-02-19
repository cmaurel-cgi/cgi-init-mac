# Documentation setup macOS (Zscaler + CrowdStrike)

Ce repo sert de base pour une documentation interne, publiee sur GitHub Pages.

## Structure
- `index.md` : page d'accueil
- `docs/` : pages de documentation

## Publier sur GitHub Pages
1. Pousser le repo sur GitHub.
2. Dans *Settings > Pages* :
   - Source : `main` / `root`.
3. Le site sera genere automatiquement.

## Travailler en local (optionnel)
Pour un rendu local, vous pouvez utiliser Jekyll :
```
# macOS
brew install ruby
bundle install
bundle exec jekyll serve
```

## Prochaine etape
Remplacer les pages dans `docs/` et `index.md` avec la vraie documentation.
