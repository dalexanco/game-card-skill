# SPEC — `print-and-play-cards`

Skill d'agent (installable via `npx skills add dalexanco/game-card-skill`) qui accompagne un utilisateur de l'idée de jeu jusqu'à des **planches A4 imprimables** : cartes de jeu (recto, verso optionnel) et planche de règles au format carte.

> La spec est en français. Le `SKILL.md` et toutes les ressources du skill sont rédigés **en anglais** ; le contenu généré (textes des cartes, règles) est produit **dans la langue de l'utilisateur**.

---

## 1. Objectifs et périmètre

### Objectifs
- Guider l'utilisateur à travers un parcours **fixe** : template → règles → design → génération des cartes → planche de règles → impression.
- Fournir des **ressources d'exemple** (templates de cartes, templates de règles, thèmes, guidelines) que l'agent ne lit **qu'à la demande** (divulgation progressive).
- Produire des fichiers HTML imprimables dont les cartes ont **la taille exacte** une fois imprimées à 100 %.
- Garantir des règles **cohérentes et équilibrées**, et une planche de règles **concise et normée**.
- Fonctionner avec **n'importe quel agent** compatible `npx skills` : la saisie se fait via un **wizard web local**, puis par chat avec l'agent, avec un **aperçu web** des planches.

### Hors périmètre (v1)
- Génération de PDF/PNG (le HTML imprimable via le navigateur suffit).
- Formats de papier autres que A4.
- Génération d'images par IA (illustrations = emojis, icônes, ou images fournies par l'utilisateur).
- Versionnement des itérations (git s'en charge si l'utilisateur le souhaite).

---

## 2. Concepts

| Terme | Définition |
| --- | --- |
| **Jeu** (*game*) | Un projet de jeu dans le dossier de l'utilisateur : config, règles, design, cartes, sorties. |
| **Itération** | Une génération complète de jeu, c'est-à-dire un passage par le parcours aboutissant à un jeu imprimable. Les itérations suivantes sur le même jeu écrasent les sorties. |
| **Template de carte** | Base HTML + CSS avec des **zones d'information** (`data-zone`). C'est un point de départ : l'agent le **copie dans le jeu** puis peut en diverger librement (ajouter, retirer, déplacer des zones, modifier le CSS). |
| **Zone d'information** | Emplacement nommé d'une carte (titre, coût, illustration, texte…). |
| **Template de règle** | Fiche décrivant une mécanique de jeu type, utilisée pour combler les trous dans les règles de l'utilisateur. |
| **Thème** | Ensemble cohérent : palette, polices, tailles, bordures, textures, ton. |
| **Tranche d'âge** | Public cible (3 tranches), qui influence règles, textes et design, indépendamment du thème. |
| **Planche** | Page A4 imprimable contenant une grille de cartes (ou la planche de règles). |

---

## 3. Arborescence

### 3.1 Dépôt

Le dépôt a deux parties :
- **les sources** (TypeScript, avec des dépendances npm), qui ne sont pas installées chez l'utilisateur ;
- **le skill distribué**, qui contient le JS **compilé et versionné**. `npx skills add` copie le dossier du skill, et il fonctionne avec `node` seul, sans `npm install`.

```
game-card-skill/
├── README.md
├── SPEC.md
├── package.json                           # deps + scripts dev/build/test (non distribué)
├── tsconfig.json
├── vite.config.ts                         # build de l'UI
├── build.mjs                              # bundle esbuild du serveur + CLI
├── src/
│   ├── core/                              # code partagé, sans I/O
│   │   ├── schema.ts                      # schémas zod : game.json, cards.json
│   │   ├── formats.ts                     # formats de carte, calcul de grille A4
│   │   ├── imposition.ts                  # placement des cartes, versos miroirs, traits de coupe
│   │   └── render.ts                      # remplissage des data-zone d'un template
│   ├── cli/                               # commandes `serve`, `build`, `validate`
│   ├── server/                            # Hono : API, fichiers statiques, SSE, watcher
│   └── ui/                                # Vite + Preact : wizard + aperçu
├── tests/                                 # vitest (core, CLI) + test Playwright des dimensions
└── skills/
    └── print-and-play-cards/
        ├── SKILL.md                       # point d'entrée : parcours + index des ressources
        ├── bin/
        │   └── ppc.mjs                    # GÉNÉRÉ : bundle unique (CLI + serveur), versionné
        ├── ui/                            # GÉNÉRÉ : build Vite (wizard + aperçu), versionné
        ├── templates/
        │   ├── cards/
        │   │   ├── INDEX.md
        │   │   ├── tcg/        { card.html, card.css, back.html, README.md }
        │   │   ├── party/      { … }
        │   │   └── resource/   { … }
        │   ├── rules/
        │   │   ├── INDEX.md
        │   │   └── <mecanique>.md          # ~12 fiches
        │   ├── themes/
        │   │   ├── INDEX.md
        │   │   └── <theme>/    { theme.css, README.md }
        │   └── layouts/
        │       └── <template>.md           # variantes de disposition par template de carte
        └── references/
            ├── print.md                   # formats, imposition, marges, traits de coupe, recto-verso
            ├── typography.md              # grille, marges de sécurité, tailles, hiérarchie
            ├── ages.md                    # règles par tranche d'âge
            ├── icons.md                   # emojis + bibliothèques d'icônes
            ├── balance.md                 # check-list de cohérence et d'équilibrage
            └── rules-sheet.md             # standard de rédaction de la planche de règles
```

Le dossier `skills/card-generator/` créé lors du scaffolding initial est renommé en `skills/print-and-play-cards/`.

### 3.1.1 Stack technique

| Besoin | Choix | Raison |
| --- | --- | --- |
| Langage | TypeScript | Types partagés entre CLI, serveur et UI via `src/core` |
| Serveur HTTP | **Hono** + `@hono/node-server` | Léger, se bundle proprement en un seul fichier. Fastify fonctionne aussi, mais il se bundle moins bien (chargements dynamiques, pino) |
| Bundle serveur/CLI | **esbuild** → `bin/ppc.mjs` (ESM, cible Node 20, toutes deps incluses) | Un seul fichier, aucun `node_modules` à l'exécution |
| UI wizard + aperçu | **Vite** + **Preact** → `ui/` | Formulaires réactifs, build statique léger |
| Validation | **zod** | Messages d'erreur exploitables par l'agent (`ppc validate`) |
| Rendu HTML côté CLI | **linkedom** | Remplit les `data-zone` et assemble les planches sans navigateur |
| Surveillance de fichiers | **chokidar** | Rechargement fiable de l'aperçu sur macOS, Linux et Windows |
| Tests | **vitest** ; **Playwright** (dev seulement) | Tests unitaires, plus une mesure des cartes en mm dans un rendu PDF |

Règles de build :
- `npm run build` régénère `skills/print-and-play-cards/bin/` et `skills/print-and-play-cards/ui/`. Les sorties sont **commitées**.
- La CI lance `npm run build && git diff --exit-code` pour refuser des sources et des sorties désynchronisées.
- `.gitattributes` marque `bin/` et `ui/` en `linguist-generated` (diffs repliés sur GitHub).
- La taille du bundle reste raisonnable (objectif < 1 Mo pour `ppc.mjs`).
- Le skill distribué ne contient ni `package.json` ni `node_modules`. Seul prérequis chez l'utilisateur : **Node ≥ 20**.

### 3.2 Projet de jeu (chez l'utilisateur)

Créé par l'agent à l'étape 0 dans `./games/<slug>/` (relatif au répertoire courant de l'utilisateur) :

```
games/<slug>/
├── game.json          # options saisies dans le wizard + choix validés (source de vérité)
├── rules.md           # règles complètes (document de travail)
├── design.md          # intention de design : thème, palette, polices, disposition
├── cards.json         # données des cartes (généré par l'agent)
├── rules-sheet.json   # texte de la planche de règles, par section imposée (§9.1)
├── template/          # copie du template de carte, modifiable librement par l'agent
│   ├── card.html
│   ├── card.css
│   ├── back.html
│   └── theme.css
└── print/
    ├── cards.html     # planches de cartes (rectos, + versos miroirs si activés)
    └── rules.html     # planche de règles
```

Les fichiers de `print/` sont **autonomes** (données et CSS intégrés, Google Fonts par lien) : ils s'impriment aussi sans le serveur.

---

## 4. Parcours (ordre fixe)

L'ordre est imposé. Chaque étape se termine par une **validation explicite** de l'utilisateur, enregistrée dans `game.json` (`steps.<étape>.validated = true`). On peut revenir à une étape précédente pour la modifier ; les étapes suivantes sont alors marquées à revalider.

### Étape 0 — Initialisation
1. Demander le nom du jeu, puis créer `games/<slug>/` et un `game.json` minimal.
2. Lancer le serveur : `node <skill>/bin/ppc.mjs serve --game games/<slug>` (port par défaut `4747`, configurable avec `--port`).
3. Donner l'URL du wizard à l'utilisateur.

### Étape 1 — Sélection du template de carte
- Le wizard affiche les **3 templates** avec un aperçu rendu (données d'exemple) et le choix du **format de carte**.
- L'agent lit `templates/cards/INDEX.md`, puis **uniquement** le `README.md` du template choisi.
- En sortie : le template est copié dans `games/<slug>/template/`.

### Étape 2 — Règles du jeu
- Le wizard recueille les règles grossières : nombre de joueurs, durée visée, **tranche d'âge**, objectif, déroulement d'un tour, système de points, condition de fin, et un champ libre.
- En chat, l'agent identifie les **trous**. Pour les combler, il lit `templates/rules/INDEX.md`, propose 1 à 3 mécaniques pertinentes, et ne lit que la ou les fiches retenues.
- L'agent applique `references/balance.md` (voir §8) et `references/ages.md`, puis présente les incohérences et déséquilibres avec des corrections proposées.
- En sortie : `rules.md` validé, avec la composition du deck (types de cartes, quantités).

### Étape 3 — Design
- Le wizard propose les **5 thèmes** (aperçu sur le template choisi), la palette (modifiable), les polices, les tailles, les variantes de disposition, l'activation du **verso** et le **mode coloriage** (§10.1.1).
- L'agent lit `templates/themes/INDEX.md`, puis le thème choisi, `templates/layouts/<template>.md`, `references/typography.md`, et `references/icons.md` si des icônes sont demandées.
- En sortie : `design.md` et `template/theme.css`. L'agent peut adapter `card.html` et `card.css` à cette étape.

### Étape 4 — Génération des cartes (itération)
- L'agent écrit `cards.json` à partir de `rules.md` et `design.md` : textes, emojis/icônes, quantités.
- L'agent lance `node <skill>/bin/ppc.mjs validate --game games/<slug>`, corrige les erreurs signalées, puis lance `build`, qui génère `print/cards.html` (voir §6). L'aperçu se recharge automatiquement.
- L'agent ne doit **jamais** écrire `print/*.html` à la main : l'imposition et les dimensions viennent exclusivement du moteur.
- On itère ensuite en chat : retouches de cartes, de template ou de thème. Chaque régénération écrase `print/`.

### Étape 5 — Planche de règles
- L'agent rédige les règles selon `references/rules-sheet.md` (voir §9) dans `rules-sheet.json` (une entrée par section imposée), puis lance `build`, qui génère `print/rules.html`.
- Le support est choisi automatiquement (voir §9.3).

### Étape 6 — Impression
- L'agent rappelle les consignes d'impression (§6.4) et liste les fichiers produits.

---

## 5. Serveur local (wizard + aperçu)

### 5.1 Contraintes
- Exécuté depuis le bundle `bin/ppc.mjs` (Hono, voir §3.1.1) : Node ≥ 20, sans installation.
- Écoute sur `127.0.0.1` uniquement.
- Sert l'UI précompilée depuis `<skill>/ui/`.
- Agnostique de l'agent : **toute communication passe par les fichiers du jeu**. L'agent lit et écrit `game.json` et les autres fichiers ; le wizard lit et écrit `game.json` via l'API.

### 5.2 Routes

| Route | Rôle |
| --- | --- |
| `GET /` | Redirige vers `/wizard` |
| `GET /wizard` | UI du wizard (étapes 1 à 3), pré-remplie depuis `game.json` |
| `GET /preview` | Aperçu : onglets « Cartes », « Versos », « Règles », zoom, bascule « carte seule / planche » |
| `GET /api/game` | Renvoie `game.json` |
| `PUT /api/game` | Fusionne et enregistre les champs saisis dans `game.json` |
| `GET /api/templates` | Liste des templates, thèmes, formats et tranches d'âge disponibles (pour le wizard) |
| `GET /game/*` | Sert les fichiers du jeu (`print/`, `template/`…) |
| `GET /events` | Server-Sent Events : notifie un changement de fichier dans le jeu, pour le rechargement automatique de l'aperçu |

### 5.3 Wizard
- Un écran par étape (1, 2, 3), avec un bouton « Enregistrer ». Le wizard indique ensuite à l'utilisateur de **revenir dans le chat** (« dites à l'agent que c'est prêt »).
- Les champs texte sont courts : les textes longs se discutent en chat.
- Le wizard ne génère rien. Il enregistre uniquement des options.

### 5.4 `game.json` (schéma indicatif)

```json
{
  "name": "Les Lapins de l'Espace",
  "slug": "lapins-espace",
  "language": "fr",
  "steps": { "template": { "validated": true }, "rules": { "validated": false }, "design": {}, "cards": {}, "rulesSheet": {} },
  "card": { "template": "tcg", "format": "poker" },
  "rules": {
    "players": { "min": 2, "max": 4 },
    "durationMinutes": 20,
    "age": "7-12",
    "objective": "…",
    "turn": "…",
    "scoring": "…",
    "endCondition": "…",
    "notes": "…",
    "mechanics": ["set-collection"]
  },
  "design": {
    "theme": "cartoon",
    "palette": { "primary": "#…", "secondary": "#…", "accent": "#…", "background": "#…", "text": "#…" },
    "fonts": { "title": "Luckiest Guy", "body": "Nunito" },
    "layout": "art-top",
    "coloring": false,
    "back": { "enabled": true }
  }
}
```

---

## 6. Impression et imposition

### 6.1 Formats de carte

| Clé | Nom | Dimensions | Grille sur A4 |
| --- | --- | --- | --- |
| `poker` | Poker (défaut) | 63 × 88 mm | 3 × 3 = 9 |
| `bridge` | Bridge | 57 × 89 mm | 3 × 3 = 9 |
| `tarot` | Tarot | 70 × 120 mm | 2 × 2 = 4 |
| `mini` | Mini euro | 44 × 68 mm | 4 × 4 = 16 |

Le calcul de grille est fait par le moteur d'imposition (nombre max de colonnes et de lignes dans la zone imprimable). Il ne doit pas être codé en dur.

### 6.2 Mise en page A4
- `@page { size: A4; margin: 0 }`. Les marges sont gérées dans la page (`10 mm` minimum de chaque côté), pour rester dans la zone imprimable des imprimantes domestiques.
- Toutes les dimensions sont en **mm** (pas de px) pour garantir la taille réelle à l'impression.
- La grille de cartes est centrée et les cartes sont **jointives** : un seul trait de coupe entre deux cartes.
- **Traits de coupe** dans la marge, prolongeant les lignes de découpe (épaisseur 0,25 pt, gris).
- **Règle de calibration** de 50 mm sur chaque planche, pour vérifier l'échelle après impression.
- Pas de fond perdu en v1. Les fonds de carte s'arrêtent au trait de coupe, et la **zone de sécurité** (§7.2) protège le contenu.
- Les quantités de `cards.json` sont développées en exemplaires avant l'imposition.

### 6.3 Versos (option désactivable)
- Si `design.back.enabled` est actif, chaque planche de rectos est suivie de sa planche de versos.
- Les versos sont **en miroir horizontal** (retournement bord long) : la colonne 1 du recto correspond à la colonne N du verso.
- Un seul dos commun par deck en v1 (`template/back.html`).

### 6.4 Consignes d'impression (affichées dans l'aperçu et rappelées par l'agent)
- Imprimer à **100 % / taille réelle**, sans « ajuster à la page ».
- Recto-verso : **retournement sur le bord long**.
- Vérifier la règle de calibration (50 mm).
- Papier conseillé : 250–300 g/m², ou papier 80 g/m² glissé dans des protège-cartes.

---

## 7. Templates de carte

### 7.1 Les 3 templates de départ

| Clé | Usage | Zones (`data-zone`) |
| --- | --- | --- |
| `tcg` | Combat, créatures, sorts | `title`, `cost`, `art`, `type`, `effect`, `flavor`, `stats.attack`, `stats.defense` |
| `party` | Questions, défis, actions | `category`, `prompt` (grande zone de texte), `points`, `icon` |
| `resource` | Ressources, valeurs, symboles | `corner.value` (×2, dont un tourné à 180°), `symbol` (central), `effect` (petit) |

Chaque template comprend :
- `card.html` : la structure. Une zone = un élément `data-zone="<nom>"`. Les zones absentes des données sont masquées.
- `card.css` : la mise en page, qui n'utilise **que** des variables CSS de thème (aucune couleur ni police en dur).
- `back.html` : le verso par défaut (motif + nom du jeu).
- `README.md` : intention, liste des zones avec longueur de texte conseillée par format, exemples de données, et pistes de divergence.

L'agent peut librement modifier sa copie dans `games/<slug>/template/`. Il doit seulement respecter les guidelines d'impression et de typographie.

### 7.2 Variables de thème (contrat entre template et thème)

```css
:root {
  /* couleurs */
  --color-primary; --color-secondary; --color-accent;
  --color-background; --color-surface; --color-text; --color-text-muted; --color-border;
  /* typographie */
  --font-title; --font-body; --font-number;
  --size-title; --size-body; --size-small; --size-number;   /* en pt */
  --line-height-body;
  /* forme */
  --radius; --border-width; --frame-padding;                /* en mm */
  --stroke-width;                                           /* traits d'illustration/ornements, en pt (voir §10.1.1) */
  /* texture (optionnel) */
  --texture-background;
  /* grille */
  --grid-unit;                                              /* ex. 1mm */
  --safe-margin;                                            /* ex. 3mm */
}
```

### 7.3 Variantes de disposition (`templates/layouts/<template>.md`)
Trois variantes par template, décrites en texte et sous forme de schémas ASCII (ce ne sont pas des fichiers complets). Par exemple pour `tcg` : illustration en haut, illustration pleine carte avec texte en surimpression, cadre épais style « carte ancienne ».

---

## 8. Règles et équilibrage

### 8.1 Templates de règles (`templates/rules/`)

Douze mécaniques : `deck-building`, `trick-taking` (plis), `set-collection`, `draft`, `push-your-luck` (stop ou encore), `bluff`, `cooperative`, `duel`, `party-quiz`, `shedding` (vider sa main), `memory`, `auction`.

Structure imposée de chaque fiche :
1. **Résumé** (2–3 phrases)
2. **Joueurs / durée / âge conseillé**
3. **Déroulement d'un tour**
4. **Système de points**
5. **Condition de fin** (et départage)
6. **Types de cartes nécessaires** (avec les zones de template conseillées)
7. **Répartition type du deck** (tableau type × quantité, pour un deck de référence)
8. **Pièges d'équilibrage**
9. **Variantes et combinaisons** avec d'autres mécaniques

`INDEX.md` contient une ligne par fiche : nom, description d'une ligne, tags (`players:2-4`, `duration:short`, `complexity:low`, `age:7+`).

### 8.2 Check-list de cohérence et d'équilibrage (`references/balance.md`)
L'agent **doit** vérifier, à l'étape 2 puis à l'étape 4 sur `cards.json` :
- La condition de fin est **toujours atteignable**, et la partie ne peut pas boucler indéfiniment.
- Un **départage** est prévu en cas d'égalité.
- La composition du deck est compatible avec le nombre de joueurs (assez de cartes pour la mise en place et toute la partie).
- La durée estimée correspond à la durée visée (nombre de tours × temps moyen par tour).
- Aucune carte ni stratégie n'est **dominante** : comparer le rapport coût / effet, la distribution des valeurs et la fréquence des effets forts.
- L'avantage du premier joueur est identifié et compensé si besoin.
- Chaque type de carte défini est utilisé par les règles, et chaque règle qui mentionne une carte a une carte correspondante.
- La complexité est adaptée à la tranche d'âge (§10).
- Les quantités sont cohérentes entre `rules.md`, `cards.json` et la section « Matériel » de la planche de règles.

Les problèmes sont présentés à l'utilisateur avec une correction proposée. Aucune modification silencieuse.

---

## 9. Planche de règles

### 9.1 Structure imposée (`references/rules-sheet.md`)

| Section | Contenu | Limite indicative (format carte) |
| --- | --- | --- |
| **Objectif** | Une phrase : comment on gagne | 25 mots |
| **Matériel** | Liste des cartes par type, avec quantités | liste |
| **Mise en place** | Étapes numérotées | 40 mots |
| **Tour de jeu** | Étapes numérotées, actions possibles | 80 mots |
| **Fin de partie & score** | Déclencheur, décompte, départage | 40 mots |
| **Précisions** | Cas particuliers (optionnel) | 40 mots |
| **Légende des symboles** | Uniquement si les cartes utilisent des icônes ou des emojis à sens de jeu | liste |

Règles de rédaction : présent de l'impératif, phrases courtes, un verbe d'action par étape, vocabulaire identique à celui des cartes (mêmes noms de types et de zones), aucun texte d'ambiance. Les limites sont ajustées selon la tranche d'âge (§10).

### 9.2 Gabarit
`rules.html` utilise le même thème que les cartes (`theme.css`) et une mise en page dédiée, avec une section par bloc et des titres normalisés traduits dans la langue du jeu.

### 9.3 Choix du support
1. **Carte recto-verso** au format du deck, si le texte tient (vérifié au rendu par détection de débordement dans l'aperçu).
2. Sinon, une **feuille pliée 2 × 2** : 4 panneaux au format carte par face, soit un dépliant de 2 × largeur × 2 × hauteur (par ex. 126 × 176 mm pour le poker), imprimé en recto-verso sur A4 avec traits de coupe. Le pliage ramène au format carte. `references/print.md` décrit l'ordre des panneaux, pour que la couverture (titre du jeu + Objectif) soit visible une fois plié.
3. Si cela déborde encore, l'agent **raccourcit le texte** au lieu d'agrandir le support.

---

## 10. Design : thèmes, âge, typographie, icônes

### 10.1 Thèmes (`templates/themes/`)

| Clé | Ambiance | Polices Google Fonts suggérées |
| --- | --- | --- |
| `fantasy` | Médiéval, parchemin, enluminures | Cinzel (titres), Crimson Text (corps) |
| `retro` | Pixel art, arcade 8-bit | Press Start 2P (titres), VT323 (corps) |
| `scifi` | Futuriste, HUD, néons | Orbitron (titres), Exo 2 (corps) |
| `cartoon` | Contours épais, couleurs franches | Luckiest Guy (titres), Nunito (corps) |
| `kawaii` | Pastels, arrondis, mignon | Mochiy Pop One (titres), M PLUS Rounded 1c (corps) |

Chaque thème fournit :
- `theme.css` : toutes les variables de §7.2, y compris les **polices et tailles** (pas uniquement les couleurs).
- `README.md` : intention, 2 à 3 palettes alternatives, polices de repli, choix d'emojis et d'icônes cohérents, bordures et textures, et ce qu'il faut éviter.

`INDEX.md` contient une ligne par thème : description et tranches d'âge compatibles.

### 10.1.1 Mode coloriage
Le mode coloriage est une **option de rendu combinable avec tous les thèmes** (`design.coloring: true`). Ce n'est pas un thème séparé. Les cartes sont imprimées **au trait** pour être coloriées à la main : aucune forme n'est remplie.

- **Activation** : l'imposition ajoute `data-mode="coloring"` sur la racine des planches. Chaque `theme.css` **doit** fournir un bloc `[data-mode="coloring"] { … }` qui redéfinit ses variables.
- **Couleurs** : fonds et surfaces en blanc (`--color-background`, `--color-surface` → `#fff`), et tous les fonds remplis, dégradés et textures (`--texture-background`) désactivés. Seuls les traits et le texte restent, en noir ou en gris très foncé.
- **Traits** : nouvelle variable `--stroke-width`, d'au moins 0,5 pt (au moins 1 pt pour la tranche `≤6`). Les zones à colorier sont **fermées** et assez grandes (au moins 5 mm pour `≤6`).
- **Typographie** : texte de jeu en noir plein, pour rester lisible. Pour les titres, une police à contour ou un texte détouré (`-webkit-text-stroke` avec un remplissage transparent) est possible, mais jamais pour le texte de jeu ni pour les valeurs numériques.
- **Icônes et emojis** : uniquement des versions au trait. Les emojis passent en police monochrome **Noto Emoji** (au lieu de Noto Color Emoji). Côté icônes : Lucide, ou Phosphor en graisse `regular`/`light`. Game-icons.net donne des silhouettes pleines : il est exclu de ce mode, sauf rendu en contour.
- **Versos** : également au trait (motif à colorier).
- **Wizard** : une case « Mode coloriage » à l'étape Design, avec l'aperçu mis à jour en direct.
- **Thème** : le `README.md` de chaque thème décrit son rendu en mode coloriage (motifs de bordure au trait, cadres, ornements).

### 10.2 Tranches d'âge (`references/ages.md`)
L'âge est un **axe indépendant du thème**. Il s'applique aux règles, aux textes et au design.

| Tranche | Règles | Textes de carte | Design |
| --- | --- | --- | --- |
| `≤6` (moins de 6 ans) | 1 mécanique, tour en 1–2 actions, pas de calcul au-delà de 10 | quasi aucun texte ; emojis ou icônes porteurs du sens | corps ≥ 14 pt, grands pictos, couleurs très contrastées, coins arrondis |
| `7-12` | 1–2 mécaniques, calculs simples | phrases ≤ 12 mots | corps ≥ 10 pt |
| `12-99` | libre | concis mais libre | corps ≥ 7 pt |

L'âge pilote aussi les limites de mots de la planche de règles (−50 % pour `≤6`, avec des pictogrammes conseillés).

### 10.3 Guidelines typographie et impression (`references/typography.md`)
- **Zone de sécurité** : aucun texte ni élément important à moins de 3 mm du bord de coupe.
- **Grille** : positionnement sur une grille de 1 mm, avec des gouttières internes constantes (multiples de 2 mm).
- **Hiérarchie** : 3 niveaux maximum (titre / corps / petit) et **2 polices maximum** (+ une police de chiffres si besoin).
- **Tailles minimales** : voir §10.2. Les valeurs numériques jouables (coût, points) doivent être lisibles d'un coup d'œil (≥ 14 pt).
- **Lignes** : épaisseur ≥ 0,25 pt ; éviter les filets fins sur fond coloré.
- **Contraste** : texte/fond conforme au WCAG AA minimum. Le texte courant est noir ou très foncé, jamais en couleur saturée claire.
- **Couleurs à l'impression** : les couleurs très saturées (RVB néon) ternissent sur une imprimante domestique ; prévoir des palettes qui restent lisibles en noir et blanc.
- **Cohérence** : une même information occupe toujours la même zone sur toutes les cartes d'un type.
- **Débordement** : réduire le texte en priorité, puis la taille de police d'au plus un cran, jamais sous le minimum de la tranche d'âge.

### 10.4 Icônes et emojis (`references/icons.md`)
- **Emojis en priorité.** Pour un rendu identique sur toutes les machines, utiliser la police Google **Noto Color Emoji**.
- Bibliothèques d'icônes gratuites disponibles sur npm, de la plus sérieuse à la plus fun. Elles sont référencées par CDN (jsDelivr/unpkg, version figée) ou en SVG intégré :
  1. **Lucide** (`lucide-static`) : sobre et lisible, licence ISC.
  2. **Phosphor** (`@phosphor-icons/core`) : 6 graisses, dont *fill* et *duotone*, plus expressif ; licence MIT.
  3. **Game-icons.net** (`@iconify-json/game-icons`) : plus de 4 000 icônes orientées jeu (épées, potions, dés…) ; licence CC BY 3.0, **attribution obligatoire** (à mentionner sur la planche de règles).
- Le fichier précise une règle d'usage : une seule bibliothèque par jeu, en plus des emojis.

---

## 11. `SKILL.md`

Contenu attendu (en anglais, concis) :
- Frontmatter : `name: print-and-play-cards`, et une `description` qui déclenche le skill sur « créer un jeu de cartes », « print and play », « cartes imprimables », « planche de cartes »…
- Le parcours des étapes 0 à 6, avec pour chaque étape : ce que l'utilisateur fait dans le wizard, ce que l'agent fait en chat, les fichiers produits, et le critère de validation.
- **Table des ressources** : quel fichier lire, et à quel moment. Règle explicite : lire un `INDEX.md` avant tout fichier de son dossier, puis seulement les fichiers choisis.
- Les commandes du CLI :

  | Commande | Rôle |
  | --- | --- |
  | `ppc serve --game <dir> [--port 4747]` | Wizard + aperçu |
  | `ppc validate --game <dir>` | Valide `game.json`, `cards.json` et `rules-sheet.json` (zod), plus les contrôles de cohérence automatisables (quantités, zones inconnues, débordements estimés) |
  | `ppc build --game <dir>` | Génère `print/cards.html` et `print/rules.html` |
  | `ppc init --game <dir> --name <nom>` | Crée le squelette du jeu |

- Si Node ≥ 20 est absent, l'agent le signale à l'utilisateur. Le skill ne prévoit pas de génération manuelle des planches de repli.
- Les règles impératives : tailles en mm, zone de sécurité, check-list d'équilibrage, structure de la planche de règles.

---

## 12. Critères d'acceptation

1. `npx skills add dalexanco/game-card-skill --list` liste `print-and-play-cards`.
2. Dans une copie du skill installée par `npx skills add` (sans `node_modules`), `node bin/ppc.mjs serve --game games/test` démarre ; `/wizard` et `/preview` s'affichent.
2bis. `npm run build && git diff --exit-code` passe : les sorties versionnées sont à jour.
3. Une saisie dans le wizard est enregistrée dans `game.json`, et une modification de `print/cards.html` recharge l'aperçu.
4. Imprimée à 100 % sur A4, une carte `poker` mesure 63 × 88 mm (±0,5 mm), et la règle de calibration mesure 50 mm.
5. Pour chaque format, le nombre de cartes par planche correspond au tableau §6.1.
6. Avec les versos activés, rectos et versos s'alignent en recto-verso bord long (miroir correct).
7. Chaque template rend correctement avec chacun des 5 thèmes, sans couleur ni police codée en dur dans `card.css`.
7bis. En mode coloriage, chaque combinaison template × thème ne contient aucun fond rempli autre que blanc (vérifié par un test Playwright sur les `background-color`/`fill` calculés), et les emojis utilisent Noto Emoji.
8. La planche de règles contient exactement les sections de §9.1, dans l'ordre, et tient sur une carte recto-verso ou sur un pliage 2 × 2.
9. Un parcours complet (de bout en bout) sur un jeu d'exemple produit un jeu jouable dont la check-list §8.2 passe.

---

## 13. Points ouverts

- **Tranches d'âge** : les bornes données (« moins de 6 ans », « 7-12 », « 12-99 ») laissent un trou à 6 ans et un chevauchement à 12 ans. La spec retient `≤6`, `7-12`, `12-99` (12 ans → `7-12`). À confirmer.
- **Plusieurs dos** : un dos par type de carte (pioche ou défausse distincte) est-il utile dès la v1 ?
- **Images utilisateur** : faut-il un emplacement `games/<slug>/assets/` pour des illustrations fournies, référencées par la zone `art` ?
