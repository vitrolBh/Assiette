# Assiette — suivi des calories et macronutriments

App web mobile-first (PWA) en **français**, 100 % statique, hébergée sur GitHub Pages :
https://vitrolbh.github.io/Assiette/

## Architecture

- **`index.html`** — TOUTE l'app (HTML + CSS + JS vanilla, zéro framework, zéro build).
  Sections du fichier, dans l'ordre : tokens CSS → layout → pages (Accueil, Journal,
  Tendances, Réglages) → sheets (ajout repas, analyse IA, manuel, check-in) → JS
  (état/stockage → objectifs → navigation → rendus → ajout repas → API Claude →
  tendances/graphes SVG → réglages → import/export → init).
- **`sw.js`** — service worker (cache offline). ⚠️ **À chaque modification de
  l'app, incrémenter la constante `CACHE`** (`assiette-v1` → `assiette-v2`, …)
  sinon les utilisateurs gardent l'ancienne version.
- `manifest.webmanifest`, `icon-192.png`, `icon-512.png`, `apple-touch-icon.png` — PWA.

## Données (localStorage, clé `assiette_v1`)

```js
{
  settings: { apiKey, model, goals: { mode: "manual"|"profile"|"default",
              manual:{kcal,prot,gluc,lip}, profile:{sexe,age,taille,poids,act,obj} },
              withings: null | {access,refresh,expiresAt,userid,lastSync} },
  meals: [ { id, date:"YYYY-MM-DD", time:"HH:MM", type, source, name,
             kcal, prot, gluc, lip, ingredients[], thumb } ],
  days:  { "YYYY-MM-DD": { poids, poidsSrc:"manuel"|"withings", pas, kcalOut, exercice } },
  favorites: [ { id, name, kcal, prot, gluc, lip, ingredients[], thumb } ]
}
```

- Les données ne quittent JAMAIS l'appareil, sauf : appels à l'API Anthropic (analyse
  repas) et échange de tokens OAuth avec le Worker Cloudflare `assiette-withings`
  (voir `WITHINGS.md`) — le Worker ne stocke rien, les tokens restent en localStorage.
- Ne jamais logger ni exposer `settings.apiKey` ni les tokens Withings. Ne rien envoyer
  vers d'autres serveurs.
- `days[d].poidsSrc` distingue une pesée saisie à la main ("manuel", prioritaire, jamais
  écrasée par une sync) d'une pesée importée automatiquement ("withings").
- Si le schéma évolue : écrire une migration depuis `assiette_v1` (ne pas perdre les données).

## API Claude (analyse des repas)

- `fetch` direct vers `https://api.anthropic.com/v1/messages` avec les en-têtes
  `x-api-key`, `anthropic-version: 2023-06-01` et
  `anthropic-dangerous-direct-browser-access: true` (requis en navigateur).
- Le modèle répond en JSON strict (plat, ingrédients, total, confiance, questions[]) —
  voir `SYS_PROMPT`. La conversation est conservée dans `state.add.convo` pour les
  tours de clarification.

## Design

- Design tokens en variables CSS (`:root` + mode sombre via `prefers-color-scheme`).
- Couleurs des séries : protéines `--prot` (bleu), glucides `--gluc` (orange),
  lipides `--lip` (vert). Ordre fixe, ne pas recycler ces couleurs pour autre chose.
- Graphiques : SVG maison (pas de lib). Grilles fines, barres à bouts arrondis 4px,
  légende dès 2 séries, libellés d'axe en gris `--muted`, `tabular-nums` pour les chiffres.
- Toute l'UI en français, ton simple et direct.

## Dev & déploiement

- Test local : `python3 -m http.server 8000` puis http://localhost:8000
  (le service worker ne s'active qu'en https ou localhost).
- Déploiement : `git push` sur `main` → GitHub Pages republie automatiquement (~1 min).
- Vérifier sur mobile (viewport 390px) et dans les deux thèmes clair/sombre.

## Roadmap (idées validées avec Ben)

- ✅ Repas favoris (réajout en 1 tap depuis la feuille d'ajout).
- ✅ Sync Withings automatique du poids (OAuth + relais Cloudflare Worker —
  voir `WITHINGS.md` pour l'architecture complète).
- Import export Apple Santé / Strava pour pas et calories dépensées.
- Rappels, scan code-barres (OpenFoodFacts) : à discuter.
