# Intégration Withings — poids synchronisé automatiquement

## Architecture

```
iPhone (Assiette, GitHub Pages)
   │  1. bouton « Connecter Withings » → redirection OAuth vers Withings
   │  2. retour sur l'app avec ?code=…
   ├──► Cloudflare Worker (relais, gratuit)  ──► API Withings
   │      /token    : échange le code contre access+refresh tokens
   │      /refresh  : renouvelle le token expiré
   │      /measure  : récupère les pesées (getmeas)
   ▼
localStorage : tokens + poids fusionnés dans db.days[date].poids
```

Pourquoi un relais : l'échange de tokens exige le `client_secret` et l'API
Withings n'envoie pas d'en-têtes CORS — un navigateur ne peut pas l'appeler
directement. Le Worker garde le secret côté serveur et ajoute le CORS
restreint à `https://vitrolbh.github.io`.

## Étape A — Créer l'app développeur Withings (Ben, ~5 min)

1. Va sur **developer.withings.com** → connecte-toi avec ton compte Withings
   (celui de l'app Health Mate).
2. Dashboard → **Create an application** → intégration « Public API ».
3. Remplis :
   - Application name : `Assiette`
   - Description : suivi personnel de nutrition
   - **Callback URL** : `https://vitrolbh.github.io/Assiette/` (exactement, avec le / final)
4. Note le **Client ID** et le **Client Secret** (⚠️ le secret ne se colle
   QUE dans Cloudflare, jamais dans le code de l'app ni dans le repo).

## Étape B — Créer le Worker Cloudflare (Ben, ~10 min, sans terminal)

1. Crée un compte gratuit sur **dash.cloudflare.com**.
2. Menu **Workers & Pages** → **Create** → **Create Worker** → nomme-le
   `assiette-withings` → **Deploy** (le code par défaut, on le remplace après).
3. Clique **Edit code**, remplace tout par le code ci-dessous, **Deploy**.
4. Onglet **Settings → Variables and Secrets** du worker, ajoute deux
   **secrets** : `WITHINGS_CLIENT_ID` et `WITHINGS_CLIENT_SECRET` (valeurs de l'étape A).
5. Note l'URL du worker : `https://assiette-withings.<ton-sous-domaine>.workers.dev`

### Code du Worker (complet)

```js
const ORIGIN = "https://vitrolbh.github.io"; // seule origine autorisée

export default {
  async fetch(req, env) {
    const cors = {
      "Access-Control-Allow-Origin": ORIGIN,
      "Access-Control-Allow-Methods": "POST, OPTIONS",
      "Access-Control-Allow-Headers": "content-type",
      "content-type": "application/json",
    };
    if (req.method === "OPTIONS") return new Response(null, { headers: cors });
    if (req.method !== "POST")
      return new Response('{"error":"POST uniquement"}', { status: 405, headers: cors });

    const path = new URL(req.url).pathname;
    const body = await req.json().catch(() => ({}));

    if (path === "/measure") {
      // Proxy des pesées — le token d'accès vient du téléphone
      const r = await fetch("https://wbsapi.withings.net/measure", {
        method: "POST",
        headers: {
          Authorization: "Bearer " + body.access_token,
          "Content-Type": "application/x-www-form-urlencoded",
        },
        body: new URLSearchParams({
          action: "getmeas",
          meastype: "1",          // 1 = poids (kg)
          category: "1",          // mesures réelles
          lastupdate: String(body.since || 0),
        }),
      });
      return new Response(await r.text(), { headers: cors });
    }

    // /token et /refresh : échange OAuth avec le client_secret
    const form = new URLSearchParams({
      action: "requesttoken",
      client_id: env.WITHINGS_CLIENT_ID,
      client_secret: env.WITHINGS_CLIENT_SECRET,
    });
    if (path === "/token") {
      form.set("grant_type", "authorization_code");
      form.set("code", body.code);
      form.set("redirect_uri", body.redirect_uri);
    } else if (path === "/refresh") {
      form.set("grant_type", "refresh_token");
      form.set("refresh_token", body.refresh_token);
    } else {
      return new Response('{"error":"introuvable"}', { status: 404, headers: cors });
    }

    const r = await fetch("https://wbsapi.withings.net/v2/oauth2", {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: form,
    });
    return new Response(await r.text(), { headers: cors });
  },
};
```

## Étape C — Côté app (à faire par Claude Code dans le repo)

Constantes à ajouter (le client_id est public, pas le secret) :

```js
const WITHINGS = {
  clientId: "<CLIENT_ID>",
  worker: "https://assiette-withings.<sous-domaine>.workers.dev",
  redirect: "https://vitrolbh.github.io/Assiette/",
  scope: "user.metrics",
};
```

1. **Réglages → carte « Sources de données »** : remplacer la ligne
   « Withings — Phase 2 » par un bouton **Connecter Withings** (puis état
   « Connecté ✓ » + bouton Déconnecter + « Synchroniser maintenant »).
2. **Connexion** : rediriger vers
   `https://account.withings.com/oauth2_user/authorize2?response_type=code&client_id=…&scope=user.metrics&redirect_uri=…&state=<aléatoire>`
   (stocker `state` dans localStorage et le vérifier au retour).
3. **Retour OAuth** : au chargement de l'app, si l'URL contient `?code=` et
   que `state` correspond → `POST {worker}/token` avec `{code, redirect_uri}`
   → la réponse Withings est `{status:0, body:{access_token, refresh_token,
   expires_in, userid}}` → stocker dans `db.settings.withings`
   (`{access, refresh, expiresAt, userid}`) → nettoyer l'URL
   (`history.replaceState`) → lancer une première sync.
4. **Sync** (au démarrage si connecté + bouton manuel) :
   - si `Date.now() > expiresAt - 60s` → `POST /refresh` d'abord
     (⚠️ Withings renvoie un NOUVEAU refresh_token à chaque fois — toujours
     remplacer les deux tokens stockés).
   - `POST /measure` avec `{access_token, since: lastSync}` ;
   - réponse : `body.measuregrps[]`, chaque groupe a `date` (timestamp unix)
     et `measures[]` ; le poids = `value * 10^unit` (ex. 78500 × 10⁻³ = 78,5 kg) ;
   - fusionner : `db.days[dateISO].poids = poids` (ne pas écraser une saisie
     manuelle plus récente le même jour) ; mémoriser `lastSync` ; toast
     « X pesées importées ».
5. **Erreurs** : toute réponse avec `status !== 0` = erreur Withings (si 401
   → retenter après refresh ; si le refresh échoue → repasser en
   « Déconnecté » avec un toast clair).
6. Incrémenter la version du cache dans `sw.js` et mettre à jour CLAUDE.md
   (section Roadmap : Withings ✅).

## Tests

- Connexion complète depuis l'iPhone (Safari) ET depuis l'app installée.
- Pesée test sur la balance → « Synchroniser maintenant » → le poids
  apparaît dans Journal et Tendances.
- Attendre l'expiration (3 h) ou forcer `expiresAt = 0` → vérifier le refresh.
- Vérifier qu'aucun secret n'apparaît dans le repo (`git grep -i secret`).

## Vigilance

- La doc officielle est sur developer.withings.com/api-reference — vérifier
  les noms de champs exacts si quelque chose renvoie `status != 0`
  (codes fréquents : 401 token invalide, 601 trop de requêtes).
- Le callback Withings doit correspondre EXACTEMENT au `redirect_uri` envoyé.
- Données : les tokens restent dans le localStorage du téléphone ; le worker
  ne stocke rien.
