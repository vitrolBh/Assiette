# Assiette — Module Entraînement (feature prioritaire)

À implémenter AVANT le module Coach (`COACH.md`), qui consommera ces données.
Respecter `CLAUDE.md` : vanilla JS, un seul `index.html`, design tokens CSS,
tout en français, incrémenter le cache `sw.js`, aucun secret dans le repo.

> **Note design** : Ben redessinera l'UI plus tard. Donc : aucune couleur en dur,
> réutiliser les classes existantes (`.card`, `.tile`, `.chip`, `.bar`, `.setrow`)
> et les variables (`--prot`, `--gluc`, `--lip`, `--surface`, `--ink`…). Toute
> nouvelle couleur = un nouveau token défini dans les DEUX thèmes.

---

## 0. Contexte athlète (à stocker, pas à coder en dur)

Ben, deux horizons simultanés :
- **Phase actuelle : reprise post-blessure** (lombaires, protocole d'un médecin
  du sport démarré fin août 2026). Priorité absolue : pas de rechute.
- **Horizon suivant : trail long** (50–100 km, gros D+). Change tout : volume,
  D+, glucides/h à l'effort, sorties longues.

Conséquence produit : le module doit gérer une **progression par phases**, pas un
objectif unique. Et il ne doit JAMAIS contredire un protocole médical en cours
(voir § 8 Garde-fous).

---

## 1. Modèle de données (ajouts)

```js
db.settings.strava = { access, refresh, expiresAt, athleteId } | null

db.settings.sport = {
  phase: "reprise" | "construction" | "specifique" | "affutage" | "coupure",
  contraintes: "texte libre (ex. lombaires, protocole médecin depuis 31/08/2026)",
  objectifs: [                                   // 0..n, triés par date
    { id, titre, type:"trail"|"route"|"forme"|"reprise",
      date_cible:"YYYY-MM-DD"|null, distance_km:null, denivele_m:null, notes:"" }
  ],
  rpeRappel: true                                // proposer de noter le RPE après import
}

db.workouts = {                                  // clé = id Strava (string)
  "<id>": { date:"YYYY-MM-DD", debut:"HH:MM", type:"Run|Hike|WeightTraining|…",
            nom, duree_min, distance_km, denivele_m, kcal, fc_moy, fc_max,
            relative_effort,                     // objectif (Strava, nécessite FC)
            rpe,                                 // subjectif 1–10 (Strava ou saisi ici)
            rpeSrc:"strava"|"app"|null, notes, source:"strava" }
}

db.plan = {                                      // clé = semaine ISO "2026-W36"
  "<semaine>": { aucunPlan:false, source:"photo"|"manuel",
                 seances:[{ jour:"YYYY-MM-DD", type, description,
                            duree_min, intensite:"facile"|"modere"|"dur"|"tres_dur" }] }
}

db.days["YYYY-MM-DD"] += {                       // s'ajoute au check-in existant
  forme: 1..5,                                   // ressenti général du jour
  sommeil_h: number|null,
  douleurs: [ { zone:"lombaires|genou_d|…", intensite:0..10, contexte:"repos|effort|apres" } ]
}

db.meals[].intraEffort = true|false              // § 7
```

**Migration** : ces clés peuvent être absentes des données existantes. Toujours
lire avec un défaut (`db.workouts = db.workouts || {}`), jamais planter.

---

## 2. Connexion Strava (base technique)

Reprendre **tel quel** le § 3 de `FEATURES-V2.md` (routes worker Cloudflare
`/strava/token`, `/strava/refresh`, `/strava/activities`, `/strava/activity`,
secrets `STRAVA_CLIENT_ID` / `STRAVA_CLIENT_SECRET`, scope `activity:read_all`,
Callback Domain `vitrolbh.github.io`).

⚠️ **Point critique** : Withings et Strava reviennent sur la MÊME URL avec
`?code=`. Préfixer le `state` (`withings_…` / `strava_…`), router le retour selon
ce préfixe, et **vérifier que Withings fonctionne toujours** après cette étape.

**Champs à extraire** de chaque activité (détail `/strava/activity`) :
`name, sport_type, start_date_local, moving_time, distance, total_elevation_gain,
calories, average_heartrate, max_heartrate`, l'effort objectif
(`suffer_score`, exposé aussi sous le nom `relative_effort` selon les clients) et
le RPE (`perceived_exertion`, 1–10, uniquement sur ses propres activités).

> ⚠️ `perceived_exertion` et `suffer_score` ne sont pas garantis présents (RPE non
> saisi, ou activité sans cardio). **Ne jamais faire dépendre une fonctionnalité
> de leur présence** : prévoir le fallback du § 3.

**Agrégation jour** : `db.days[iso].kcalOut` = somme des `calories` des activités
du jour ; `exercice = true`. Ne pas écraser une saisie manuelle plus élevée.

---

## 3. RPE : « c'était dur, mais est-ce que c'était VRAIMENT dur ? »

Le besoin nº1 de Ben. Principe : croiser le **ressenti** (RPE 1–10, subjectif) et
l'**effort mesuré** (Relative Effort de Strava, calculé sur la FC).

**Saisie** : après chaque import, si une activité n'a pas de `perceived_exertion`,
afficher une pastille « RPE ? » sur la carte de la séance → sélecteur 1–10 avec
libellés (1 très facile … 5 modéré … 8 dur … 10 maximal). Stocker `rpe` + `rpeSrc:"app"`.

**Lecture** (calcul local, déterministe — pas d'IA) : pour chaque séance, comparer
le RPE à l'effort mesuré normalisé sur **les 90 derniers jours de séances
comparables** (même famille de sport, durée du même ordre) :

- z-RPE = rang percentile du RPE parmi les séances comparables
- z-RE  = rang percentile du Relative Effort parmi les mêmes séances
- écart = z-RPE − z-RE

Affichage en une phrase, sans jargon :
- écart > +25 pts → « Tu l'as vécue plus dure que ce que dit ton cardio. Fatigue,
  sommeil, chaleur ou nutrition peuvent expliquer l'écart. »
- écart < −25 pts → « Séance plus exigeante physiologiquement que ressentie —
  attention à ne pas sous-estimer la récupération. »
- sinon → « Ressenti cohérent avec l'effort mesuré. »

**Conditions d'affichage** : au moins **8 séances comparables** dans la fenêtre,
sinon afficher « pas encore assez d'historique pour comparer » (honnêteté avant tout).
Ne rien afficher si `relative_effort` absent (pas de cardio) — proposer alors juste
le RPE brut et sa moyenne mobile.

---

## 4. Plan de la semaine (Campus Coach → photo)

**Campus Coach n'a pas d'API publique** ; en revanche l'app exporte déjà ses
séances vers les montres Garmin nativement — **on ne réimplémente donc PAS
l'export montre**, on l'assume comme hors périmètre et on le dit à l'utilisateur
dans un texte d'aide (« l'envoi vers ta montre se fait depuis Campus Coach »).

**Import du plan** : page Entraînement → « Plan de la semaine » → deux boutons :
- **📷 Importer depuis une capture** : l'utilisateur screenshote sa semaine dans
  Campus Coach, l'app envoie l'image à Claude (même mécanique que les repas, prompt
  système dédié) qui renvoie un JSON strict :
  `{"semaine":"2026-W36","seances":[{"jour":"YYYY-MM-DD","type":"...","description":"...","duree_min":0,"intensite":"facile|modere|dur|tres_dur"}]}`.
  Écran de validation avant enregistrement (l'utilisateur corrige/supprime).
- **Pas de plan cette semaine** : bouton qui pose `aucunPlan:true`. **Important** :
  dans ce cas le coach ne doit JAMAIS parler d'écart au plan — juste de charge et
  de progressivité.

**Rapprochement plan ↔ réalisé** : matcher par date + type (tolérance ±1 jour).
Afficher par séance : ✅ conforme · ⚠️ plus long/dur que prévu · ➖ non réalisée.
Le rapprochement est un **calcul local**, pas une interprétation IA.

---

## 5. Journal douleur & forme

Étendre la feuille « Check-in » (déjà existante) avec, sous un séparateur :
- **Forme du jour** : 5 chips (1 = épuisé → 5 = très bien).
- **Sommeil** : champ heures (optionnel).
- **Douleur** : bouton « + Ajouter une douleur » → zone (liste : lombaires, cervicales,
  genou G/D, cheville G/D, mollet G/D, ischios G/D, hanche G/D, pied G/D, épaule,
  autre) + intensité 0–10 (slider) + contexte (au repos / à l'effort / après l'effort).
  Plusieurs douleurs possibles par jour ; suppression possible.

**Visualisation** (page Entraînement) : une frise sur 30/90 jours superposant
l'intensité de douleur (par zone, une ligne par zone active) et la charge
hebdomadaire. Objectif : voir d'un coup d'œil si la douleur suit les pics de charge.
Palette : réutiliser les tokens existants + un token `--pain` (rouge du statut
critique déjà défini) ; jamais une couleur en dur.

---

## 6. Analyse croisée « nutrition × charge × ressenti »

C'est le cœur de la demande de Ben — et l'endroit où il faut être **rigoureux
intellectuellement**, parce que c'est là qu'une app peut raconter n'importe quoi.

### 6a. Ce que l'app calcule (local, déterministe)

Un objet `buildInsightContext()` qui produit, pour la fenêtre demandée :

**Charge**
- volume hebdo : minutes, km, D+, nombre de séances
- somme des Relative Effort sur 7 j (charge aiguë) et moyenne des blocs de 7 j sur
  28 j (charge chronique) ; ratio aigu/chronique
- variation du volume semaine N vs N−1 (%), et vs moyenne des 4 semaines

**Nutrition** (hors intra-effort, cf. § 7)
- moyennes 7 j / 28 j : kcal, P/G/L en g et en % de l'apport
- écart moyen aux objectifs
- balance énergétique estimée : ingéré − (dépense activité) — **en précisant que
  le métabolisme de base n'est pas mesuré**, donc c'est un indicateur relatif
- régularité : nombre de jours renseignés / total (fiabilité de l'échantillon)

**Ressenti & santé**
- moyennes de forme, de sommeil, jours avec douleur, intensité max/moyenne par zone
- poids : tendance sur la fenêtre (pente de régression, en kg/semaine)

**Qualité des données** : `joursRenseignes`, `joursAvecRepas`, `joursAvecCheckin`,
`seancesAvecRPE`, `seancesAvecFC`. Ces chiffres accompagnent TOUJOURS l'analyse.

### 6b. Ce que l'IA fait avec

Bouton « Analyser ma période » (page Entraînement, 7 / 28 / 90 jours). L'app envoie
le contexte ci-dessus + les objectifs sportifs + les contraintes, et demande :

```json
{"resume":"2-3 phrases sur l'état général",
 "observations":[{"constat":"...","donnees":"les chiffres qui l'appuient","confiance":"faible|moyenne|bonne"}],
 "hypotheses":[{"piste":"...","test":"ce que Ben pourrait observer/essayer 2 semaines pour vérifier"}],
 "actions":[{"quoi":"...","pourquoi":"..."}],
 "vigilance":["signaux à surveiller"]}
```

**Règles imposées au modèle (dans le prompt système, non négociables)** :
1. **Jamais de causalité.** Écrire « on observe que… », « une piste possible… »,
   jamais « X a causé Y ». Une corrélation sur quelques semaines de données d'une
   seule personne ne démontre rien.
2. **Toujours situer la fiabilité** : citer le nombre de jours réellement
   renseignés ; si < 14 jours de données exploitables, dire explicitement que
   c'est trop tôt pour dégager quoi que ce soit et s'en tenir à des constats bruts.
3. **Proposer un test**, pas un verdict : chaque hypothèse s'accompagne d'une façon
   simple de la vérifier sur 2 semaines (c'est ce qui rend l'outil utile).
4. **Pas de diagnostic médical**, pas de nom de pathologie, pas de posologie.
5. **Ne jamais contredire un protocole médical** mentionné dans `contraintes` :
   en cas de tension entre une suggestion et le protocole, le protocole gagne et
   le modèle invite à en parler au professionnel qui suit Ben.
6. Ton direct et concret, en français, tutoiement, pas de flatterie.

### 6c. Garde-fous « drapeaux rouges » (calculés localement, AVANT l'IA)

Si l'une de ces conditions est vraie, afficher un encart d'orientation **au-dessus**
de toute analyse, et interdire au modèle de proposer une adaptation d'entraînement :
- douleur ≥ 7/10 un jour quelconque de la fenêtre ;
- douleur au repos ou nocturne signalée ≥ 3 jours ;
- même zone douloureuse ≥ 5 jours sur les 7 derniers ;
- intensité d'une zone en hausse sur 14 jours (pente positive).

Texte de l'encart (à reprendre tel quel) : « Cette douleur dure ou s'intensifie.
L'app peut t'aider à suivre, pas à décider : parles-en à ton médecin du sport ou
à ton kiné avant d'adapter quoi que ce soit. »

---

## 7. Nutrition intra-effort

Choix retenu : **catégorie séparée, comptée dans les calories, exclue des ratios.**

- Ajouter `intra-effort` comme 5ᵉ type de repas (chips de la feuille d'ajout, icône ⚡).
  Tout repas de ce type porte `intraEffort:true`.
- **Totaux du jour** : deux niveaux désormais.
  - `totalsOf(date)` (inchangé, tout compris) sert aux calories et aux graphes.
  - `totalsOf(date, {horsIntra:true})` sert au calcul des **ratios de macros**, aux
    barres de progression des macros et à tout ce que reçoit le coach.
- **Affichage Accueil / Journal** : sous les barres de macros, une ligne discrète
  « dont ⚡ 36 g de glucides à l'effort » quand il y en a.
- **Coach & analyses** : ne jamais reprocher un pic de glucides intra-effort ;
  au contraire, savoir l'utiliser (« sur ta sortie longue tu étais à 30 g/h,
  c'est bas pour un format trail long »).
- **Suggestion automatique** : si une activité Strava du jour dure > 75 min et
  qu'aucun repas intra-effort n'est enregistré, proposer (toast discret, non
  bloquant) « Tu as pris quelque chose pendant ta sortie ? » → ouvre l'ajout
  pré-réglé sur intra-effort.

---

## 8. Page « Entraînement » (nouvel onglet)

La barre passe à 5 onglets : Accueil · Journal · **Entraînement** · Tendances · Réglages.
(Si 5 onglets serrent trop sur petit écran, fusionner Journal et Tendances plus tard —
à voir au redesign ; ne pas casser la navigation existante.)

Contenu, de haut en bas :
1. **Cette semaine** : tuiles volume (min), D+, séances, charge (somme RE) + comparaison
   à la semaine précédente.
2. **Plan de la semaine** : liste des séances prévues vs réalisées (§ 4), ou état
   « Pas de plan cette semaine ».
3. **Séances récentes** : cartes (date, nom, durée, D+, kcal, RE, RPE + pastille de
   saisie du RPE si manquant, et la phrase ressenti/mesuré du § 3).
4. **Forme & douleurs** : frise 30 j (§ 5) + accès rapide au check-in.
5. **Analyse** : sélecteur 7/28/90 j + bouton « Analyser ma période » (§ 6), résultat
   en cartes ; cache par (fenêtre, jour) pour ne pas rappeler l'API inutilement.
6. **Objectifs** : liste des objectifs sportifs + phase en cours (édition dans Réglages).

**Réglages** : nouvelle carte « Sport » → phase, contraintes (champ libre),
objectifs (ajout/édition/suppression : titre, type, date cible, distance, D+, notes).

---

## 9. Ce qu'on ne fait PAS (et pourquoi)

- **API Garmin directe** : réservée aux partenaires sous contrat. Contournement :
  Garmin Connect → Strava en synchro automatique (à activer une fois côté Garmin).
- **Export de séance vers la montre** : Campus Coach le fait déjà nativement ;
  fabriquer des fichiers d'entraînement à sa place serait fragile et redondant.
- **API Campus Coach** : inexistante publiquement → import par capture d'écran.
- **Génération de séances de rééducation** : hors périmètre tant qu'un protocole
  médical est en cours (§ 6c). L'app adapte le volume/l'intensité, elle ne soigne pas.

---

## 10. Ordre de livraison (un commit par étape, test entre chaque)

1. Connexion Strava + import des séances + agrégation kcalOut (§ 2)
2. Page Entraînement minimale : semaine + séances récentes (§ 8.1, 8.3)
3. RPE : saisie + comparaison ressenti/mesuré (§ 3)
4. Nutrition intra-effort (§ 7) — touche les totaux, tester soigneusement
5. Journal douleur & forme + frise (§ 5)
6. Plan de la semaine par photo (§ 4)
7. Analyse croisée + garde-fous (§ 6) — **les garde-fous du § 6c AVANT l'appel IA**
8. Réglages Sport & objectifs (§ 8)

## 11. Tests

- Withings fonctionne toujours après l'ajout du retour OAuth Strava (les deux
  connexions, dans les deux ordres).
- Refresh du token Strava après expiration (6 h) — le refresh_token change à chaque fois.
- Une activité sans cardio (marche téléphone) : pas de Relative Effort → l'app ne
  doit rien casser ni afficher de comparaison.
- Un jour avec un gel intra-effort : kcal du jour inchangées, ratios de macros
  recalculés hors gel, mention « dont ⚡ ».
- Fenêtre d'analyse avec < 14 jours de données → le texte doit dire que c'est trop tôt.
- Douleur 8/10 saisie → l'encart d'orientation apparaît et aucune adaptation n'est proposée.
- Mode clair et sombre, iPhone 390 px, app installée depuis l'écran d'accueil.
