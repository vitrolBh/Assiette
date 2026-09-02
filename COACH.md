# Assiette — Module Coach nutrition

À implémenter **après** `ENTRAINEMENT.md` : le coach consomme les séances, la
charge, le RPE, les douleurs et la nutrition intra-effort. Mêmes règles de style
et de design que `CLAUDE.md` (tokens, français, pas de couleur en dur).

---

## 1. Deux modes, un seul moteur

| Mode | Déclenchement | Question à laquelle il répond |
|---|---|---|
| **À la demande** | Bouton sur l'Accueil | « Qu'est-ce que je mange maintenant ? » |
| **Bilan du soir** | Automatique, à l'ouverture après une heure configurable | « Comment s'est passée ma journée, et demain ? » |

Le mode à la demande est **prospectif** (que faire du reste de la journée), le
bilan du soir est **rétrospectif + prépare le lendemain**. Deux prompts distincts,
un seul constructeur de contexte.

> **Sur l'automatisation réelle** : une notification poussée sur iPhone demanderait
> un service de push (VAPID + serveur) — faisable via le worker Cloudflare, mais
> c'est un chantier à part. **V1 : pas de notification.** Le bilan est généré à la
> première ouverture de l'app après l'heure choisie, et une pastille discrète
> signale qu'il est disponible. Si Ben veut la vraie notification plus tard, ce
> sera une étape dédiée (documentée en § 8).

---

## 2. Réglages — nouvelle carte « Coach »

- **Régime** (chips, choix unique → `db.settings.diet`) : omnivore · végétarien ·
  pescétarien · vegan · flexitarien · high-protein · keto · méditerranéen.
- **Allergies / aversions** (champ libre → `db.settings.dietNotes`) — texte injecté
  tel quel dans le prompt comme contrainte absolue.
- **Bilan du soir** : interrupteur `db.settings.coachSoir` + heure (défaut 21 h).
- **Style du coach** (chips → `db.settings.coachTon`) : factuel · encourageant ·
  cash. Défaut : factuel. (Utile pour le redesign : le ton fait partie du produit.)
- **Compléments alimentaires** (→ `db.settings.complements`) : liste libre, saisie
  manuelle ou extraite d'une photo d'étiquette (Ben prend une cure personnalisée
  Cuure). Voir § 6b.

---

## 3. Contexte envoyé au modèle

Réutiliser `buildInsightContext()` (§ 6a d'`ENTRAINEMENT.md`) et y ajouter le
présent :

```js
{
  moment: { date, heure, jour_semaine },
  objectifs_jour: { kcal, prot, gluc, lip },
  consomme_jour: { kcal, prot, gluc, lip },          // HORS intra-effort
  intra_effort_jour: { kcal, gluc },                  // à part, jamais reproché
  restant: { kcal, prot, gluc, lip },                 // peut être négatif
  repas_du_jour: [ { type, nom, heure, kcal, P, G, L } ],
  semaine: { plats: [noms des 7 derniers jours], moyennes: {…}, jours_renseignes },
  sport: {
    aujourdhui: [ { type, duree_min, kcal, denivele_m, relative_effort, rpe } ],
    demain_prevu: [ … ] | null,                        // depuis db.plan
    charge_7j, charge_28j, ratio,
    phase, objectifs
  },
  sante: { forme_jour, sommeil_h, douleurs_actives, poids_tendance_kg_sem },
  regime, allergies, complements, ton
}
```

Toujours **hors intra-effort** pour les macros : un gel de 36 g de sucre pendant
une sortie longue n'est pas une erreur alimentaire, c'est du carburant.

---

## 3bis. Profil d'habitudes alimentaires

Le coach doit connaître **comment Ben mange réellement**, pas un modèle théorique.
Répartition du travail : l'app calcule ce qui est chiffrable (heures, macros,
régularité), le modèle repère ce qui est sémantique (les aliments récurrents,
les schémas). Ne pas tenter de normaliser les noms de plats en JS — c'est fragile
et le modèle le fait mieux à partir des noms bruts.

`buildHabitsProfile()` sur les **60 derniers jours** :

```js
{
  parType: {                         // petit-déjeuner, déjeuner, dîner, collation
    "<type>": { heureMediane:"08:40", occurrences: 42, joursCouverts: 0.86,
                kcalMedian: 420, splitMedian: { prot:0.22, gluc:0.48, lip:0.30 } }
  },
  semaine:   { kcalMoy, splitMoy, repasParJour },   // lundi→vendredi
  weekend:   { kcalMoy, splitMoy, repasParJour },   // samedi→dimanche
  regularite: { joursAvecAuMoins2Repas: 0.78, joursRenseignes: 47 },
  platsRecents: [ { nom, date, type, kcal, prot, gluc, lip } ]  // 30 derniers jours, noms BRUTS
}
```

Note : `heureMediane` par type sert au § 4bis. `platsRecents` est la matière
première de l'analyse sémantique — envoyer les noms tels que saisis.

---

## 4bis. Suggestion adaptée au moment

Le type de repas suggéré ne doit **pas** venir de seuils horaires figés mais des
habitudes réelles : si Ben déjeune habituellement à 13 h 15 et qu'il est 14 h,
c'est encore le déjeuner ; s'il grignote d'ordinaire vers 16 h 30, à 16 h c'est
une collation.

Règle : pour chaque type, une fenêtre = `heureMediane ± 90 min`. Le type retenu est
celui dont la fenêtre contient l'heure courante ; si plusieurs correspondent,
celui qui n'a pas encore été enregistré aujourd'hui ; si aucun, « collation ».
Moins de 5 occurrences pour un type → retomber sur les seuils par défaut.

Le contexte du § 3 est complété par : `moment.typeSuggere`, `moment.estWeekend`,
`habitudes` (le profil ci-dessus). Le prompt du § 4 gagne trois règles :

- **Ancrer dans ses habitudes** : proposer d'abord des choses proches de ce qu'il
  mange déjà (ses classiques reviennent dans `platsRecents`), avec une variante
  qui corrige ce qui manque. Une suggestion crédible est une suggestion suivie.
- **Adapter au type et au jour** : un petit-déjeuner reste un petit-déjeuner ;
  le week-end autorise plus de temps de préparation que le mardi midi.
- **Ne jamais suggérer un repas déjà pris** aujourd'hui, sauf s'il reste beaucoup
  de marge et que l'heure s'y prête.

---

## 5bis. Revue de la semaine (analyse des aliments)

C'est la fonction « oh, tu as pris un pain suisse quatre fois cette semaine ».
Déclenchée à la demande (bouton dans la carte Coach : **« Ma semaine »**) et
proposée automatiquement dans le bilan du dimanche soir.

Entrée : `platsRecents` (7 jours), les moyennes de la semaine, les objectifs,
le régime, et le contexte sportif. Sortie JSON :

```json
{"resume":"1-2 phrases sur la semaine",
 "aliments_marquants":[{"aliment":"...","occurrences":0,
   "composition":"ce qu'il apporte, en macros","effet":"ce que ça change pour ses objectifs",
   "alternative":"option concrète à composition proche mais mieux alignée"}],
 "habitudes":[{"constat":"...","donnees":"les chiffres qui l'appuient"}],
 "a_essayer":"une seule chose à tester la semaine prochaine"}
```

### Règles de ton — la partie délicate

L'objectif est un coach **utile et direct**, pas un juge. Concrètement :

**Ce que le modèle FAIT :**
- **Décrire la composition, factuellement** : « un pain suisse, c'est environ
  400 kcal, essentiellement glucides et lipides, 6 g de protéines. »
- **Nommer la fréquence** : « quatre fois cette semaine, c'est devenu ton
  petit-déjeuner par défaut » — c'est le schéma qui compte, pas l'écart isolé.
- **Relier aux objectifs, pas à la morale** : « tes matins pèsent lourd en
  calories et te laissent loin de ta cible protéines, que tu finis par courir
  après le soir. »
- **Donner l'alternative concrète et comparable** : « pour à peu près les mêmes
  calories, un skyr + granola + banane t'apporte ~25 g de protéines en plus. »

**Ce que le modèle NE FAIT PAS :**
- Pas de « bon » ni de « mauvais » aliment, pas de « c'est dommage », pas de
  « tu as craqué », pas d'aliment « interdit », pas de compensation par le sport.
- Pas de commentaire sur un aliment isolé mangé une fois : en dessous de
  **3 occurrences sur 7 jours**, un aliment n'entre pas dans `aliments_marquants`.
- Pas de leçon sur le plaisir alimentaire : un aliment apprécié qui revient
  raisonnablement n'est pas un problème à résoudre.

**Pourquoi cette limite** (à garder même si le ton « cash » est sélectionné) :
un outil qui fait culpabiliser sur un petit-déjeuner finit désinstallé, et
surtout, associer culpabilité et nourriture est contre-productif — y compris
pour la performance. Le ton « cash » rend le propos plus direct et plus bref,
il ne rend pas le jugement moral acceptable. Ce qui doit piquer, c'est la
précision du constat, pas la réprobation.

---

## 4. Prompt « Que manger maintenant ? »

Sortie JSON strict :

```json
{"lecture":"1-2 phrases : où j'en suis aujourd'hui",
 "suggestions":[{"plat":"...","kcal":0,"prot_g":0,"gluc_g":0,"lip_g":0,
                 "pourquoi":"lien explicite avec le restant / la séance du jour",
                 "effort":"rapide|moyen|cuisine"}],
 "a_eviter":"optionnel : ce qui ferait dépasser une limite, dit sans moraliser"}
```

Règles du prompt système :
- **Respect absolu** du régime et des allergies. Aucune exception, aucune
  suggestion « tu pourrais faire une entorse ».
- **Raisonner sur le restant**, pas sur l'idéal théorique. Si les lipides sont déjà
  au max et qu'il reste 40 g de protéines : proposer maigre (blanc de poulet,
  cabillaud, skyr, tofu ferme, légumineuses selon régime), pas une omelette au fromage.
- **Tenir compte du sport du jour** : après une grosse séance, prioriser protéines +
  glucides de récupération ; jour de repos, on peut serrer les glucides.
- **Tenir compte de l'heure** : pas de plat complet à 15 h, pas de dîner à 22 h 30.
- **Varier** par rapport aux plats de la semaine (ils sont dans le contexte).
- 2 à 3 suggestions, dont au moins une « rapide » (< 10 min).
- Portions réalistes et chiffrées (« 150 g de cabillaud + 200 g de riz »).
- Si les objectifs sont déjà atteints : le dire simplement et suggérer léger ou rien.
  **Ne jamais pousser à manger plus pour « remplir » un objectif.**

**Bouton « + Ajouter comme repas »** sur chaque suggestion → pré-remplit la saisie
manuelle (nom + 4 macros), l'utilisateur valide ou ajuste. **Ne jamais enregistrer
un repas suggéré automatiquement** : ce qui est mangé est déclaré par Ben, pas
supposé par l'app.

---

## 5. Prompt « Bilan du soir »

```json
{"bilan":"2-3 phrases sur la journée (nutrition + sport, liés)",
 "points_forts":["..."],
 "ajustements":[{"quoi":"...","pourquoi":"..."}],
 "demain":"1-2 phrases orientées par la séance prévue (ou son absence)",
 "qualite_donnees":"phrase si des repas manquent manifestement"}
```

Règles supplémentaires :
- **Pas de bilan sur des données trop incomplètes** : si < 2 repas enregistrés,
  dire « journée peu renseignée, je ne peux pas en tirer grand-chose » plutôt que
  d'inventer une analyse sur du vide.
- **Aucun jugement moral** sur la nourriture : pas de « bon/mauvais aliment », pas de
  « tu as craqué », pas de compensation (« tu devras courir pour rattraper »). On
  parle d'apports et d'objectifs, jamais de mérite ou de culpabilité.
- Si les apports sont très en dessous des besoins plusieurs jours d'affilée,
  le signaler **comme un risque de sous-alimentation** (surtout en reprise
  post-blessure : la consolidation demande de l'énergie et des protéines), et non
  comme une réussite.
- Reprendre les garde-fous « drapeaux rouges » d'`ENTRAINEMENT.md` § 6c.

---

## 6. Micronutriments : ce qu'on peut honnêtement faire

Ben a demandé un suivi des micronutriments. Deux niveaux, il faut choisir le bon :

- ❌ **Ce qu'on ne fait pas** : afficher « 12 mg de fer, 68 % des apports
  recommandés ». Sans base de données alimentaire (type Ciqual/USDA) et sans pesée
  précise, ces chiffres seraient inventés avec 3 chiffres significatifs de fausse
  précision. C'est exactement le genre de nombre auquel on finit par croire.
- ✅ **Ce qu'on fait** : une **couverture qualitative** sur 7 jours. À partir des
  noms de plats et ingrédients de la semaine, le modèle estime la présence des
  grandes familles : fibres, légumes verts, fruits, oméga-3, fer, calcium,
  légumineuses, produits fermentés. Rendu sous forme de 3 niveaux
  (bien couvert / moyen / peu vu cette semaine), **jamais de chiffre**, avec la
  mention « estimation à partir de tes descriptions, pas une analyse nutritionnelle ».

```json
{"couverture":[{"famille":"fibres","niveau":"bien|moyen|faible","vu":"exemples tirés de la semaine"}],
 "suggestions_ciblees":["1-2 aliments concrets pour combler ce qui manque"]}
```

Si un jour tu veux du chiffré fiable, la vraie voie est d'intégrer **OpenFoodFacts**
(base ouverte, gratuite, avec code-barres) pour les produits emballés — à mettre
dans la roadmap, pas dans ce module.

### 6b. Compléments alimentaires

`db.settings.complements` : liste libre (saisie manuelle, ou extraite d'une photo
d'étiquette via le même moteur d'analyse que les repas — Ben prend une cure
personnalisée Cuure). Format souple : `[{ nom:"Fer bisglycinate", dose:"14 mg" }]`
ou simple texte si l'extraction est approximative.

Injectée dans le contexte du coach (§ 3). Le modèle doit :
- **en tenir compte** dans la couverture du § 6a : une famille déjà complémentée
  n'est pas signalée comme manquante — au mieux mentionnée (« ton fer passe surtout
  par la complémentation, l'alimentaire reste léger ») ;
- **ne jamais recommander** d'ajouter un complément, d'en changer, ni de modifier
  une dose. Ce n'est ni son rôle ni son domaine — les suggestions restent
  alimentaires. Si une carence semble se dessiner, il oriente vers un professionnel.

---

## 7. UI

**Accueil** — sous la carte des macros, une carte « Coach » :
- État par défaut : bouton **« Que manger maintenant ? »** + une ligne de contexte
  (« il te reste 620 kcal et 38 g de protéines »).
- Après appel : `lecture` en tête, puis les suggestions en cartes (plat, macros,
  pourquoi, bouton + Ajouter), et un lien « ↻ Autres idées ».
- Si le bilan du soir est prêt : la carte affiche d'abord le bilan, avec le bouton
  de suggestion en dessous.

**Page Entraînement** — le bloc « Analyser ma période » (§ 6 d'`ENTRAINEMENT.md`)
reste le lieu de l'analyse longue durée. Le coach de l'accueil, lui, reste court :
3 cartes maximum, lisible en 10 secondes.

---

## 8. Coût, cache et évolutions

- **Aucun appel automatique** hors bilan du soir (1/jour max).
- Cache : `db.coachCache = { date, type, contexteHash, result }`. Réutiliser le
  résultat si même jour, même type, et contexte inchangé (hash des repas + séances).
  « Autres idées » force un nouvel appel.
- La **revue de la semaine** (§ 5bis) se met en cache par semaine ISO : un appel
  par semaine suffit, sauf demande explicite de rafraîchissement.
- Modèle : Haiku suffit largement pour les suggestions (rapide, économique) ;
  Sonnet pour le bilan du soir et l'analyse de période. Rendre ça configurable
  plutôt que de le figer.
- **Plus tard, si Ben le veut** : vraie notification du bilan → Web Push (iOS 16.4+
  pour une PWA installée), ce qui suppose des clés VAPID, un abonnement stocké et
  un déclencheur planifié dans le worker Cloudflare. Étape séparée, pas en V1.

---

## 9. Limites à afficher dans l'app (une fois, dans la carte Coach)

Une ligne discrète, permanente : « Suggestions générées par IA à partir de tes
saisies. Ce n'est pas un avis médical ni diététique. »

Et dans les Réglages, sous la carte Coach : « Les estimations de macros viennent
d'un modèle de langage : elles sont utiles pour une tendance, pas exactes au gramme. »

---

## 10. Tests

- Régime vegan sélectionné → aucune suggestion animale, jamais (tester plusieurs relances).
- Allergie « arachides » → aucune suggestion en contenant, ni sauce satay ni beurre de cacahuète.
- Lipides déjà dépassés à midi → les suggestions restent maigres.
- Jour avec sortie longue + gel intra-effort → le coach ne reproche pas les glucides
  et parle de récupération.
- Journée avec 0 ou 1 repas → refus argumenté de faire un bilan, pas d'invention.
- Objectifs déjà atteints → ne pousse pas à manger davantage.
- Cache : deux clics successifs = un seul appel API ; « Autres idées » = nouvel appel.
- **Habitudes** : à 16 h, si Ben grignote habituellement à cette heure-là, la
  suggestion proposée est une collation — pas un dîner.
- **Revue de la semaine** : un aliment mangé une seule fois n'apparaît pas dans
  `aliments_marquants` ; un aliment revenu 4 fois est relevé, avec sa composition
  et une alternative — et sans aucun « c'est dommage » ni « tu as craqué »
  (tester aussi avec le ton « cash » sélectionné).
