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
  regime, allergies, ton
}
```

Toujours **hors intra-effort** pour les macros : un gel de 36 g de sucre pendant
une sortie longue n'est pas une erreur alimentaire, c'est du carburant.

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
