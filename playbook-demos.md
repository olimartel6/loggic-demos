# Playbook — Créer une démo de site web + une démo d'app de fidélité pour un commerce local (Québec)

> Document de transfert. À donner tel quel à un agent Claude Code.
> Objectif : partir d'un simple lien (Facebook, Instagram, site web ou juste un nom) et produire deux livrables prêts à envoyer en prospection, avec du **branding 100 % réel** — jamais de placeholder, jamais d'invention.

---

## 0. Ce qu'on livre

Deux artéfacts distincts, chacun avec sa propre URL publique :

| Livrable | Ce que c'est | Techno |
|---|---|---|
| **Site vitrine** | Une page unique, éditoriale, haut de gamme. Le commerce doit se reconnaître dès la première seconde. | 1 seul `index.html` + dossier `img/`. Aucune dépendance de build. |
| **App de fidélité** | Prototype interactif : points, paliers, récompenses, QR, parrainage. | Soit un `index.html` autonome, soit un tenant dans une app multi-tenant existante (voir §7). |

**Règle d'or** : le prospect doit ouvrir le lien et penser « ils ont vraiment travaillé sur *mon* commerce », pas « c'est un template ». Tout le playbook sert cette phrase.

### Le résultat

Chaque démo est **une URL publique ordinaire**. Le prospect clique, la page s'ouvre. Aucun compte, aucune installation, aucun mot de passe, aucune invitation. Ça marche sur téléphone, sur ordinateur, dans le navigateur intégré de Messenger et d'Instagram. Le lien se colle dans un DM, un courriel, un texto.

Corollaire à assumer : ces pages sont **publiques et indexables**. N'y mets rien de confidentiel. Ce sont des démos jetables de prospection — si le commerce devient client payant, on refait le site dans un dépôt privé.

---

## 0.5 Prérequis techniques

À vérifier une seule fois, avant la première démo.

| Outil | Pourquoi | Vérification |
|---|---|---|
| **Claude Code avec accès Bash** | Tout passe par des commandes shell | — |
| **Compte GitHub + `gh` authentifié** | C'est ce qui héberge les démos et génère l'URL publique | `gh auth status` |
| **`git`** | Publication | `git --version` |
| **Node.js** | Captures d'écran de vérification | `node --version` |
| **Python 3 + Pillow** | Échantillonnage des couleurs du logo | `python3 -c "import PIL"` |
| **Firecrawl** (MCP ou clé d'API) | Recherche et scraping. Fortement recommandé. | — |

Notes :

- **Le compte GitHub détermine l'URL.** Elle sera `https://<son-compte>.github.io/loggic-<slug>/`. Chaque personne publie sous son propre compte — pas besoin d'accès partagé.
- **Sans Firecrawl**, la recherche se dégrade mais reste faisable avec `WebSearch` + `WebFetch`. Les commandes `curl` du §2 (logo, photos) fonctionnent de toute façon sans aucune clé.
- **Sans Puppeteer**, l'étape de vérification visuelle saute — c'est la plus importante du playbook. Installe-le : `npm i puppeteer` dans le dossier de travail.
- **L'app de fidélité hébergée** (`demo.logiccsupplies.ca`) appartient à l'infrastructure existante. Sans ces accès, on suit la voie autonome du §4 plutôt que le §7.

---

## 1. Recherche — trouver le vrai commerce

### 1.1 Résoudre un lien de partage

Les liens `facebook.com/share/XXXX` ne disent rien. Il faut suivre les redirections :

```bash
curl -sI -L -A "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)" \
  "https://www.facebook.com/share/XXXXXXXX/" | grep -i "^location:"
```

Ça donne le vrai nom et l'ID numérique de la page — ex. `Studio-Mon-Ongle/61572509951219`. Garde l'ID, il sert plus loin.

### 1.2 Trouver les informations factuelles

**Firecrawl (et la plupart des scrapers) ne supportent pas `facebook.com` ni `instagram.com`.** N'insiste pas, contourne :

```
firecrawl_search  →  "Nom Exact Du Commerce" ville
```

Les agrégateurs indexent ce que Facebook cache. Les plus utiles :

- **wanderboat.ai** — adresse complète, téléphone, note Google, **avis Google en texte intégral**, photos de clients. C'est la mine d'or.
- **Yelp**, **Pages Jaunes**, **Google Maps** — adresse, horaires, catégorie.
- Le site officiel s'il existe — couleurs, polices, services, ton.

Scrape la page de l'agrégateur avec `firecrawl_scrape` en markdown. Tu en ressors normalement :
adresse exacte · téléphone · note et nombre d'avis · 3-5 avis verbatim · URLs de photos.

### 1.3 Ce qu'il faut absolument avoir avant de coder

- [ ] Nom **exact** (vérifié sur leur propre page — « Studio Rebel Pilates », pas « Studio Rebel »)
- [ ] Adresse complète et quartier/ville
- [ ] Téléphone réel
- [ ] Instagram / Facebook réels
- [ ] Le **positionnement** : ce qu'ils répètent eux-mêmes (une spécialité, une technique, une promesse)
- [ ] 3 avis clients verbatim
- [ ] Logo + 4 à 8 photos de leur travail

**Si une info manque, elle disparaît du livrable. On n'invente jamais** : ni horaires, ni courriel `info@domaine`, ni « 15 ans d'expérience », ni prix. Un site avec 4 sections vraies bat un site avec 8 sections dont 4 inventées — et une info fausse tue la crédibilité au moment exact où le prospect la remarque.

---

## 2. Extraction du branding réel

### 2.1 Logo — le crawler Facebook

C'est la méthode la plus fiable pour un commerce québécois. L'ID numérique de la page suffit :

```bash
curl -s -A "Googlebot/2.1 (+http://www.google.com/bot.html)" \
  "https://lookaside.fbsbx.com/lookaside/crawler/media/?media_id=<PAGE_ID>" \
  -o logo.jpg -w "%{http_code} %{size_download}\n"
```

Le user-agent Googlebot est essentiel — sans lui, Facebook renvoie une page HTML.

**Vérifie toujours le fichier** : `file logo.jpg` doit dire `JPEG image data`, pas `HTML document`. Puis **regarde-le** avec l'outil Read. Un logo contient souvent des informations qu'aucune page ne donne : le prénom du propriétaire, le quartier, un slogan, la palette exacte.

Ordre de repli si ça échoue : `og:image` du site officiel → Bing Images → CDN Yelp → photo de profil Instagram → en dernier recours, un lettrage typographique du nom (jamais un logo générique).

### 2.2 Photos du travail

Les photos d'avis Google exposées par wanderboat sont **de vraies photos prises par de vraies clientes**. Elles sont parfaites : authentiques, récentes, non retouchées.

```bash
curl -s "https://img4.boatcdn.com/review_img/<hash>" -o g1.jpg
```

**Regarde chaque photo** avant de l'utiliser. Tu cherches : le travail lui-même, l'intérieur du commerce, un objet de marque (carte d'affaires, enseigne, emballage). Tu écartes : les photos floues, les selfies, les captures d'écran, tout ce qui ne montre pas le commerce.

Une carte d'affaires photographiée par une cliente t'offre souvent le slogan officiel — impossible à trouver ailleurs.

### 2.3 Couleurs — les prendre dans le logo, pas les deviner

```python
from PIL import Image
from collections import Counter
im = Image.open('logo.jpg').convert('RGB').resize((200, 200))
for col, n in Counter(im.getdata()).most_common(10):
    print('#%02X%02X%02X' % col, n)
```

Pour isoler une couleur d'accent (dorure, cuivre, néon) que la couleur de fond écrase, filtre par canal :

```python
c = Counter(p for p in im.getdata() if p[0] > 140 and p[2] < 150 and p[0] > p[2] + 40)  # tons chauds
```

Tu construis ensuite une palette de 6 jetons :

```
--primary       la couleur dominante du logo
--primary-deep  la même, ~10 % plus sombre  (fonds, nav)
--accent        la couleur secondaire du logo (or, cuivre, rose…)
--accent-light  éclaircie          (dégradés, survols)
--cream         blanc cassé chaud  (sections claires)
--ink           gris très foncé    (texte courant, jamais du noir pur)
```

**Un seul accent.** Deux couleurs vives se battent et le résultat a l'air d'un template.

---

## 3. Design du site vitrine

### 3.1 L'ADN visuel

Ce qui distingue ces démos d'un site générique :

- **Typographie XXL.** Titre d'accueil à `clamp(3rem, 8.4vw, 6.6rem)`, en serif display à graisse légère (`Cormorant Garamond`, `Playfair Display`, `Fraunces`). Le nom du commerce doit remplir l'écran.
- **Une serif + une sans.** Serif pour les titres, sans géométrique pour le corps (`Jost`, `Outfit`, `Inter`). Jamais plus de deux familles.
- **Titres empilés sur deux lignes**, la deuxième en italique dans la couleur d'accent. C'est la signature.
- **Boutons en pilule** — `border-radius: 999px`, `letter-spacing: .18em`, texte en majuscules à `.78rem`.
- **Zéro emoji.** Nulle part. C'est un site d'entreprise.
- **Les ombres font le travail, pas les bordures.** Ombres multi-couches douces, `translateY(-6px)` au survol.
- **Espacement généreux** — `padding: clamp(5rem, 11vw, 9.5rem)` par section. Le vide, c'est le luxe.
- **Animations discrètes** : loader au chargement, nav qui se solidifie au défilement, apparition en fondu au scroll (`IntersectionObserver`, décalage de 90 ms par élément), bandeau défilant, images qui zooment lentement au survol.
- Toujours un bloc `@media (prefers-reduced-motion: reduce)` qui neutralise tout.

### 3.2 Structure des sections

```
1. Loader          nom du commerce en lettrage espacé sur fond de marque
2. Nav fixe        logo réel + liens + un CTA en pilule
3. Hero            2 colonnes : titre XXL + accroche + 2 CTA + 3 statistiques
                   | image réelle dans un cadre en arche, badge de note flottant
4. Bandeau         les services défilant en boucle
5. Le studio       2 colonnes : photos décalées | récit + 3 piliers numérotés
6. Services        grille de 6 cartes, filet dégradé qui se déploie au survol
7. Galerie         section sombre, grille 3 colonnes, vraies photos
8. Avis            3 avis Google verbatim + la note en gros chiffre
9. Contact         liste de définitions (adresse/tél/horaire/réseaux) | carte Google
10. Pied de page   copyright + réseaux
```

Adapte le nombre de sections à la matière disponible. Pas de galerie sans photos.

### 3.3 Rédaction

- **Français du Québec**, vouvoiement, accents impeccables (`œ`, `à`, `é`, `Québec`).
- Les titres de sections sont **courts et concrets** : « Un moment pour vous », « Ce qui se fait au studio », « Ce qu'en disent les clientes ».
- Le paragraphe d'accroche reprend **leur propre positionnement**, avec leur vocabulaire. Si leur page dit « builder gel — plus durable et plus doux que l'acrylique », cette phrase se retrouve dans le hero.
- Les descriptions de services décrivent la **prestation vécue**, pas une liste de mots-clés.
- **Jamais de prix.** Tu ne les connais pas, et le prix se discute en privé.
- Une statistique ne s'affiche que si elle est vérifiable. « 5,0 étoiles Google » oui. « 500 clients satisfaits » non.

### 3.4 Détails techniques

- Un seul fichier, CSS et JS en ligne. Zéro dépendance sauf Google Fonts.
- Balises `og:title` / `og:description` / `og:image` — le lien sera partagé dans Messenger et le carrousel de prévisualisation compte.
- `<img>` avec `alt` descriptif en français.
- Points de rupture à 960 px (grilles en 1 colonne, menu burger) et 520 px (boutons pleine largeur).
- Carte Google intégrée sans clé d'API : `https://www.google.com/maps?q=<adresse+encodée>&output=embed`. **Ne mets pas `loading="lazy"`** dessus, sinon elle reste vide sur les captures d'écran.

---

## 4. Design de l'app de fidélité

### 4.1 Le modèle de données

```js
{
  businessName: 'Nom Exact',
  slug: 'nom-exact',
  tagline: '<leur vrai slogan, ou une phrase tirée de leur positionnement>',
  logo: 'logos/nom-exact.png',
  heroImage: 'images/nom-exact/hero.jpg',
  galleryImages: [ /* 4 vraies photos */ ],
  pointsPerDollar: 10,      // 10 points par dollar : les chiffres restent lisibles
  referralBonus: 75,
  visitBonus: 25,
  pointsLabel: 'Points <mot de leur marque>',
  theme: { primary, primaryLight, accent, accentLight, accentDark, bg },
  rewards: [ /* exactement 4, voir plus bas */ ],
  phone, address
}
```

### 4.2 Les récompenses — c'est ici que ça se joue

Quatre récompenses, en escalier, **rédigées dans le vocabulaire du métier**. C'est le détail qui prouve qu'on a compris leur commerce.

| Palier | Rôle | Exemple — salon d'ongles |
|---|---|---|
| ~200 pts | Atteignable dès la 2ᵉ visite | Retouche ou pose de bijou offerte |
| ~400 pts | Un rabais en pourcentage | 10 % sur votre prochain rendez-vous |
| ~650 pts | Un service à valeur perçue | Nail art personnalisé offert |
| ~1000 pts | L'objectif, le service complet | Remplissage builder gel gratuit |

Calibre les paliers sur le **panier moyen du segment**. À 10 points le dollar, un salon d'ongles (~70 $ la visite = 700 pts) et un café (~6 $ = 60 pts) n'ont pas du tout la même échelle. Une récompense inatteignable démotive ; une récompense trop facile ne vaut rien.

Jamais de récompense générique (« 5 $ de rabais ») quand une récompense métier existe.

### 4.3 Écrans

```
Accueil     salutation + palier + solde en très gros sur photo réelle
            + barre de progression vers la prochaine récompense
            + 3 compteurs (achats / visites / parrainages)
            + galerie « Nos spécialités » + activité récente
Récompenses catalogue avec photos, état verrouillé/déverrouillé
Offres      promotions ponctuelles
Mon QR      code QR à faire scanner en commerce
Parrainage  code + partage
```

Paliers : Bronze 0 · Argent 500 (×1,5) · Or 2000 (×2) · Platine 5000 (×3).

### 4.4 Mode démo — pas d'inscription

L'écran d'ouverture propose **« Voir la démo »** en action principale, avec la mention « Aucun compte requis · démo interactive · 30 secondes ». La connexion existe, mais en lien secondaire discret.

C'est la leçon la plus chère du projet : **chaque formulaire avant la démo fait perdre la moitié des prospects.** Ils ne créeront pas de compte pour évaluer un outil qu'ils ne connaissent pas.

Le mode démo charge un client fictif avec un historique crédible. **Calibre les montants sur le segment** — des achats de 8 $ et 22 $ dans une démo pour un salon d'ongles trahissent immédiatement le template.

### 4.5 Style visuel

Clair et haut de gamme, jamais de mode sombre. Fond blanc cassé chaud, cartes blanches sans bordure avec ombres multi-couches, boutons en dégradé de la couleur d'accent, barres de progression de 6 px avec un reflet animé, barre de navigation basse en verre dépoli, coins à 20 px, nombres animés au chargement.

---

## 5. Vérification — avant toute livraison

Cette étape n'est pas optionnelle. Un lien mort envoyé à un prospect, c'est le prospect perdu.

```bash
# 1. Chaque URL répond 200
curl -s -o /dev/null -w "%{http_code}\n" "<url-du-site>"
curl -s -o /dev/null -w "%{http_code}\n" "<url-de-l-app>"

# 2. Chaque image aussi — un 404 d'image, c'est un trou noir dans la page
for f in logo.png hero.jpg g1.jpg g2.jpg; do
  echo -n "$f "; curl -s -o /dev/null -w "%{http_code}\n" "<base>/img/$f"
done
```

```js
// 3. Capture d'écran et REGARDE le résultat
import puppeteer from 'puppeteer';
const b = await puppeteer.launch({ args: ['--no-sandbox'] });
const p = await b.newPage();
await p.setViewport({ width: 1440, height: 900 });
await p.goto(url, { waitUntil: 'networkidle2' });
await new Promise(r => setTimeout(r, 3000));   // laisse les polices et animations finir
await p.screenshot({ path: 'top.png' });
await p.screenshot({ path: 'full.png', fullPage: true });
await b.close();
```

Ouvre les captures avec l'outil Read et cherche activement les défauts. Ceux qui reviennent le plus souvent :

- **Texte blanc sur un dégradé blanc** — un fondu de bas de section qui avale les statistiques du hero.
- **Grille orpheline** — `repeat(auto-fit, minmax(200px, 1fr))` avec 6 éléments donne 5 + 1. Fixe le nombre de colonnes.
- **Carte vide** — `loading="lazy"` sur l'iframe.
- **Images étirées** — `object-fit: cover` + `aspect-ratio` sur tous les conteneurs.
- **Débordement horizontal** sur mobile.

Refais la capture après chaque correctif. Ne déclare jamais que c'est prêt sans avoir regardé la version corrigée.

---

## 6. Déploiement

Un dépôt GitHub public par site, servi par GitHub Pages :

```bash
cd <dossier-du-site>
git init && git add -A && git commit -m "Site vitrine <Nom>"
gh repo create <compte>/loggic-<slug> --public --source=. --push
gh api repos/<compte>/loggic-<slug>/pages -X POST --input - <<< '{"source":{"branch":"main","path":"/"}}'
# → https://<compte>.github.io/loggic-<slug>/
```

La première publication prend une à trois minutes. Attends le 200 :

```bash
until curl -s -o /dev/null -w "%{http_code}" "<url>" | grep -q 200; do sleep 5; done
```

Après une mise à jour, attends de voir **ton correctif** dans le HTML servi, pas seulement un 200 — le CDN sert l'ancienne version un moment :

```bash
until curl -s "<url>" | grep -q "<un-bout-de-ton-correctif>"; do sleep 5; done
```

---

## 7. Variante : brancher sur une app de fidélité multi-tenant existante

Si tu as accès à une application de fidélité déjà déployée qui sert plusieurs commerces, tu n'écris pas d'app : tu ajoutes un tenant. Le schéma habituel :

1. Déposer le logo dans `public/logos/<slug>.png` et les photos dans `public/images/<slug>/`.
2. Ajouter l'objet de configuration (§4.1) dans le fichier de config des tenants.
3. Insérer une ligne dans la table des commerces (slug, nom, téléphone, adresse, `logo_url`, `theme` en JSON, `rewards` en JSON) avec un `on conflict (slug) do update` pour rester ré-exécutable.
4. Construire, générer les pages par tenant si l'app produit des pages avec balises Open Graph, puis publier.
5. L'URL de démo devient `https://<domaine>/t/<slug>/`.

Le point important : **une page HTML dédiée par tenant**, avec ses propres balises `og:`. Une application monopage avec `?tenant=slug` affiche la même prévisualisation pour tous les commerces dans Messenger et Snapchat, ce qui ruine l'effet de personnalisation au moment précis où le prospect reçoit le lien.

---

## 8. Erreurs à ne pas répéter

| Erreur | Pourquoi c'est grave |
|---|---|
| Image d'illustration générique | Le prospect la reconnaît en une seconde. Toute la démo perd sa crédibilité. |
| Couleur inventée | Le commerce ne se reconnaît pas. C'est un template, et ça se voit. |
| Courriel `info@domaine.com` deviné | Il rebondit. Écris `COURRIEL_À_TROUVER` et va le chercher. |
| Nom approximatif | La faute la plus visible de toutes. Copie-colle depuis leur page. |
| Prix affichés | Ni les leurs (tu les ignores), ni les tiens (ça se discute en privé). |
| Emoji dans le site | Ça casse instantanément le registre haut de gamme. |
| Mode sombre pour l'app | Rejeté à l'usage. Clair et luxueux. |
| Inscription avant la démo | Divise le taux d'ouverture par deux. |
| Livrer sans regarder une capture | Un lien cassé chez un prospect ne se rattrape pas. |
| Publier sans construire | Le site déployé reste l'ancienne version, sans avertissement. |

---

## 9. Séquence complète

```
1.  Résoudre le lien           → nom réel + ID de page
2.  Rechercher                 → adresse, téléphone, avis, positionnement
3.  Extraire le logo           → lookaside Facebook + Googlebot
4.  REGARDER le logo           → prénom, quartier, slogan, palette
5.  Extraire les photos        → photos d'avis Google, en regarder chacune
6.  Échantillonner les couleurs→ palette de 6 jetons
7.  Rédiger le site            → un index.html, leur vocabulaire
8.  Configurer l'app           → 4 récompenses calibrées au segment
9.  Déployer                   → GitHub Pages, attendre le 200
10. Capturer et REGARDER       → corriger, recapturer
11. Livrer                     → les deux liens + ce qui est réel dedans
```

L'étape la plus souvent sautée est la 10, et c'est celle qui décide si le prospect répond.

---

## 10. Exemple abouti

**Studio Mon Ongle** — salon d'ongles, Cap-Rouge (Québec). Point de départ : un seul lien de partage Facebook.

- Site : <https://olimartel6.github.io/loggic-studio-mon-ongle/>
- App : <https://demo.logiccsupplies.ca/t/studio-mon-ongle/>

Ce que la recherche a produit, et qu'aucun template n'aurait pu inventer :

- Le logo Facebook a livré la palette (marine `#2E3E4E` + or `#C9AB6D`), le prénom du propriétaire (**Micky**), le quartier (**Cap-Rouge**) et le positionnement (**service personnalisé**).
- Une cliente avait photographié la carte d'affaires : le slogan officiel, **« Où vos ongles racontent votre histoire »**, est devenu la ligne d'accroche du site.
- Leur page Facebook mettait en avant le **builder gel, plus durable et plus doux que l'acrylique** — c'est devenu l'argument central du hero et la récompense la plus haute de l'app.
- Sept vraies photos de leurs poses, trois avis Google verbatim, la note de 5,0.

Total : environ vingt minutes, dont la moitié en recherche. **C'est le bon ratio.** Un site magnifique avec du contenu générique ne convertit pas ; un site correct avec du contenu vrai convertit.
