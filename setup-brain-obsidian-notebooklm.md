# Setup — « Second cerveau » Obsidian + NotebookLM + /wrapup

> Document de transfert. À donner tel quel à un agent Claude Code, qui exécute le montage de bout en bout sur un Mac.
> Résultat : chaque session de travail se résume toute seule, atterrit dans Obsidian, se pousse dans NotebookLM, et devient interrogeable en langage naturel des mois plus tard.

---

## 0. Ce qu'on monte

Quatre pièces qui s'emboîtent :

```
   Session Claude Code
          │  /wrapup
          ▼
   Résumé de session ──────► Obsidian « Brain AI »  (vault local, iCloud)
                                     │
                                     │  brain-sync.sh, aux 2 h
                                     ▼
                             NotebookLM « AI Brain »
                                     │
                                     │  questions en langage naturel
                                     ▼
                         /briefing  ·  /weekly  ·  /ask
```

| Pièce | Rôle |
|---|---|
| **Vault Obsidian** | La source de vérité lisible. Notes, projets, idées, sessions. Synchronisé par iCloud, donc consultable sur iPhone. |
| **NotebookLM** | Le moteur de questions. On lui pose « qu'est-ce qu'on avait décidé pour X en mai ? » et il répond en citant les sources. |
| **`brain-sync.sh`** | Le pont automatique entre les deux, dans les deux sens. |
| **Skill `/wrapup`** | Ce qui alimente le tout. Sans lui, le système reste vide. |

**Le principe** : on n'écrit jamais dans NotebookLM à la main. On travaille dans Obsidian (ou on laisse `/wrapup` écrire), et la synchronisation s'occupe du reste.

---

## 1. Prérequis

```bash
# Homebrew
which brew || /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install jq python@3.12
brew install --cask obsidian     # si pas déjà installé

mkdir -p ~/bin
# S'assurer que ~/bin est dans le PATH du shell
grep -q 'HOME/bin' ~/.zshrc || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
```

Il faut aussi un **compte Google** (celui qui aura accès à NotebookLM) et **iCloud Drive activé** si on veut le vault sur iPhone.

---

## 2. Le vault Obsidian

### 2.1 Créer la structure

Ouvrir Obsidian une fois et créer un vault nommé **`Brain Ai`** dans le dossier iCloud d'Obsidian. Le chemin sera :

```
~/Library/Mobile Documents/iCloud~md~obsidian/Documents/Brain Ai
```

Puis :

```bash
VAULT="$HOME/Library/Mobile Documents/iCloud~md~obsidian/Documents/Brain Ai"
mkdir -p "$VAULT"/{Projects,Ideas,Notes,Sessions,Templates,Topics}
```

| Dossier | Contenu |
|---|---|
| `Projects/` | Une note par projet en cours. C'est la cible des `[[wiki links]]`. |
| `Topics/` | Une note par sujet transversal (un outil, une techno, une méthode). |
| `Ideas/` | Idées brutes, pas encore des projets. |
| `Notes/` | Tout le reste — références, comptes rendus, recherche. |
| `Sessions/` | **Écrit automatiquement par `/wrapup`.** Ne pas y toucher à la main. |
| `Templates/` | Gabarits Obsidian. |

### 2.2 La convention qui fait marcher le système

**Le tag `#brain` est l'interrupteur de synchronisation.** Une note qui le porte part vers NotebookLM. Une note sans le tag reste privée et locale. C'est tout le mécanisme — il n'y a pas d'autre configuration à faire par note.

Frontmatter type :

```yaml
---
tags:
  - brain
  - nom-du-projet
date: 2026-08-09
---
```

Deux règles de tenue :

1. **Toujours des tags de projet en plus de `#brain`.** Une session qui touche trois projets porte les trois tags. C'est ce qui permet de retrouver « tout ce qui concerne le client X » plus tard.
2. **Toujours des `[[wiki links]]` vers les notes de projet.** Les tags ne créent aucun lien dans la vue Graphe — seuls les wiki links le font. Terminer chaque note par une ligne :

```markdown
**Projets reliés :** [[Projects/Client-X|Client X]] · [[Topics/Supabase|Supabase]]
```

Sans ça, les sessions restent des îlots isolés et le second cerveau ne vaut rien.

### 2.3 Gabarits

Créer `Templates/Note.md`, `Templates/Projet.md`, `Templates/Idee.md`, chacun pré-rempli avec `tags: [brain]`. Activer le module **Templates** dans Obsidian (Réglages → Modules de base) et pointer le dossier de gabarits sur `Templates/`.

---

## 3. NotebookLM

### 3.1 Installer le CLI

Dans un environnement virtuel dédié — pas dans le Python du système :

```bash
python3.12 -m venv ~/.notebooklm-venv
~/.notebooklm-venv/bin/pip install --upgrade pip
~/.notebooklm-venv/bin/pip install "notebooklm-py[browser]"
~/.notebooklm-venv/bin/playwright install chromium
ln -sf ~/.notebooklm-venv/bin/notebooklm ~/bin/notebooklm
```

### 3.2 S'authentifier

```bash
notebooklm login
```

Un Chromium s'ouvre : se connecter avec le compte Google. Les cookies sont enregistrés dans `~/.notebooklm/storage_state.json`.

**Cette session expire** — comptes quelques mois. Quand les scripts commencent à échouer en silence, c'est presque toujours ça : relancer `notebooklm login`.

### 3.3 Créer le carnet

```bash
notebooklm list --json                        # voir s'il existe déjà
notebooklm create "AI Brain de Charles-Antoine" --json
```

**Garder l'identifiant retourné** — c'est un UUID du genre `5c77a4a1-...`. Il va dans le script de synchronisation, dans le skill `/wrapup`, et dans un fichier mémoire pour que les sessions futures le retrouvent seules.

Commandes utiles ensuite :

```bash
notebooklm source list --notebook <ID> --json
notebooklm source add <fichier.md> --notebook <ID> --title "Titre"
notebooklm source delete-by-title "Titre" --notebook <ID>
```

---

## 4. Le pont : `brain-sync.sh`

Script à créer dans `~/bin/brain-sync.sh`, rendu exécutable (`chmod +x`). Il fait quatre choses à chaque passage.

### 4.1 Ce qu'il fait

**a) Auto-liens.** Il parcourt les notes modifiées depuis le dernier passage et remplace les noms de projets écrits en clair par des `[[wiki links]]`. Une ligne `sed` par nom de projet :

```bash
line=$(echo "$line" | sed -E 's/(^|[^[])\bNomDuProjet\b([^]]*$|[^]])/\1[[Projects\/Nom-Du-Projet|NomDuProjet]]\2/g')
```

Il saute le frontmatter et le premier `# titre`, et ignore ce qui est déjà entre `[[ ]]`.

**Cette liste est propre à chaque personne.** Il faut la remplir avec les vrais noms de projets, de clients et d'outils. C'est cinq minutes de travail qui rendent la vue Graphe utilisable.

**b) Étiquetage automatique.** Toute note dans `Projects/`, `Ideas/` ou `Notes/` qui n'a pas `#brain` se le fait ajouter — dans la section `tags:` existante, ou avec un frontmatter neuf si elle n'en a pas. On ne peut donc pas oublier de taguer.

**c) Poussée Obsidian → NotebookLM.** Toutes les notes portant `#brain` (sauf `Sessions/` et `Templates/`) sont envoyées comme sources. Le script compare la date de modification du fichier avec celle mémorisée dans le fichier d'état : inchangée, il saute ; modifiée, il supprime l'ancienne source par son titre puis pousse la nouvelle.

**d) Rapatriement NotebookLM → Obsidian.** Les résumés de session ajoutés depuis un autre appareil sont écrits dans `Sessions/`.

### 4.2 Le fichier d'état — l'erreur à ne pas reproduire

Le script tient un fichier JSON :

```json
{ "pushed": { "note.md": {"title": "...", "file_modified": "1786..."} }, "pulled_sessions": [] }
```

**Ne pas le mettre dans le vault iCloud.** C'est le bogue qui a coûté le plus cher sur la première installation : le fichier était dans `<vault>/.obsidian/`, iCloud et Obsidian le tenaient ouvert, le `jq | mv` du script échouait sans erreur visible, et chaque exécution repoussait tout. Résultat concret : **1 335 doublons dans NotebookLM, dont 163 copies de la même note.**

Sur une installation neuve, mettre le fichier hors iCloud dès le départ :

```bash
STATE_FILE="$HOME/.brain-sync-state.json"     # PAS dans le vault
```

Et vérifier après chaque écriture que l'entrée est bien là :

```bash
tmp=$(mktemp)
jq --arg f "$filename" --arg m "$file_mod" '.pushed[$f] = {"file_modified": $m}' "$STATE_FILE" > "$tmp" && mv "$tmp" "$STATE_FILE"
jq -e --arg f "$filename" '.pushed[$f]' "$STATE_FILE" >/dev/null || log "ALERTE : état non écrit pour $filename"
```

Diagnostic si les doublons reviennent — comparer la date de modification du fichier d'état avec l'heure du dernier passage :

```bash
stat -f "%Sm" ~/.brain-sync-state.json
tail -20 ~/.brain-sync.log
```

Si le fichier n'a pas bougé depuis le dernier passage, le bogue est de retour.

### 4.3 Squelette

```bash
#!/bin/bash
set -uo pipefail

VAULT="$HOME/Library/Mobile Documents/iCloud~md~obsidian/Documents/Brain Ai"
NOTEBOOK_ID="<UUID_DU_CARNET>"
NOTEBOOKLM="$HOME/.notebooklm-venv/bin/notebooklm"
STATE_FILE="$HOME/.brain-sync-state.json"
LAST_SYNC_FILE="$HOME/.brain-sync-lastrun"
LOG_FILE="$HOME/.brain-sync.log"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >> "$LOG_FILE"; }

mkdir -p "$VAULT/Sessions"
[ -f "$STATE_FILE" ] || echo '{"pushed":{},"pulled_sessions":[]}' > "$STATE_FILE"

log "=== Début de la synchronisation ==="
# a) auto-liens   b) étiquetage   c) poussée   d) rapatriement
date '+%s' > "$LAST_SYNC_FILE"
log "=== Fin ==="
```

Toujours utiliser le **chemin absolu** vers `notebooklm` : `launchd` n'hérite pas du PATH du shell.

---

## 5. Automatisation avec launchd

### 5.1 Synchronisation aux 2 h

`~/Library/LaunchAgents/com.<user>.brain-sync.plist` :

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key><string>com.USER.brain-sync</string>
    <key>ProgramArguments</key>
    <array><string>/Users/USER/bin/brain-sync.sh</string></array>
    <key>StartInterval</key><integer>7200</integer>
    <key>RunAtLoad</key><true/>
    <key>StandardOutPath</key><string>/Users/USER/.brain-sync-stdout.log</string>
    <key>StandardErrorPath</key><string>/Users/USER/.brain-sync-stderr.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key><string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/Users/USER/bin:/Users/USER/.notebooklm-venv/bin</string>
        <key>HOME</key><string>/Users/USER</string>
    </dict>
</dict>
</plist>
```

```bash
launchctl load -w ~/Library/LaunchAgents/com.USER.brain-sync.plist
launchctl list | grep brain-sync
```

**Le bloc `EnvironmentVariables` → `PATH` n'est pas optionnel.** Sans lui, `launchd` démarre avec un PATH minimal et tous les binaires Homebrew (`jq`, `python`, `gh`) sont introuvables. Le script échoue en silence. C'est la cause numéro un des agents `launchd` qui « ne font rien ».

Sur Apple Silicon, Homebrew est dans `/opt/homebrew/bin` ; sur Intel, dans `/usr/local/bin`. Mettre les deux.

### 5.2 Rappel de `/wrapup` le soir

Même forme, avec `StartCalendarInterval` (22 h) au lieu de `StartInterval`, exécutant un script qui envoie une notification. Sans ce rappel, on oublie de faire le wrapup et le système se vide.

```xml
<key>StartCalendarInterval</key>
<dict><key>Hour</key><integer>22</integer><key>Minute</key><integer>0</integer></dict>
```

---

## 6. Le skill `/wrapup`

C'est la pièce qui alimente tout le reste. Fichier : `~/.claude/skills/wrapup/SKILL.md`.

```markdown
---
name: wrapup
description: Fin de session — résume la session, sauvegarde les mémoires clés, et pousse un journal dans le carnet NotebookLM AI Brain. Se déclenche sur "/wrapup", "wrap up", "fin de session", "résumé de session".
---
```

Le corps du skill enchaîne :

**Étape 0 — Vérifier le carnet.** Chercher l'identifiant dans les mémoires. Absent : `notebooklm list --json`, et proposer d'en créer un. Une fois créé, enregistrer l'identifiant dans un fichier mémoire pour que les sessions futures le retrouvent seules.

**Étape 1 — Relire la session.** Repérer : décisions prises et pourquoi · travail complété · apprentissages non évidents · fils ouverts · préférences de travail révélées.

**Étape 2 — Sauvegarder les mémoires.** Mettre à jour les fichiers existants plutôt que d'en créer des doublons. Ne rien sauvegarder qui soit déjà déductible du code ou de l'historique git. Convertir les dates relatives en dates absolues.

**Étape 3 — Rédiger le résumé.**

```markdown
# Résumé de session — AAAA-MM-JJ

## Ce qu'on a fait
## Décisions prises
## Apprentissages
## Fils ouverts
## Outils et systèmes touchés
```

**Étape 4 — Écrire dans Obsidian.** Dans `Sessions/session-AAAA-MM-JJ.md` (suffixe `-2`, `-3` s'il y a plusieurs sessions la même journée), avec le frontmatter `tags: [session, brain, <projets>]`, et la ligne `**Projets reliés :**` avec les wiki links. Puis ajouter le titre à `pulled_sessions` dans le fichier d'état, pour que le cron ne le rapatrie pas en double.

**Étape 5 — Pousser dans NotebookLM.**

```bash
notebooklm source add /tmp/session-AAAA-MM-JJ.md --notebook <ID>
```

**Étape 6 — Confirmer brièvement.** Combien de mémoires, résumé poussé ou non, fils ouverts pour la prochaine fois.

**Gestion d'erreur** : si l'authentification échoue, sauvegarder les mémoires localement, sauter la poussée, et **le dire**. Un wrapup qui échoue en silence, c'est une journée de travail perdue sans que personne s'en aperçoive.

---

## 7. Les skills de lecture

Une fois que le carnet se remplit, ce sont eux qui rendent le système utile au quotidien.

| Skill | Ce qu'il fait |
|---|---|
| `/briefing` (ou « bon matin ») | Lit les dernières sessions, les projets actifs et les mémoires, et produit un point du matin. |
| `/weekly` | Résumé des 7 derniers jours : progrès, objectifs, métriques. Sauvegardé dans Obsidian et poussé dans NotebookLM. |
| `/ask` | Question en langage naturel adressée au carnet. |
| `/note` | Capture rapide dans `Notes/`, avec tags et liens. |
| `/brain-sync` | Force une synchronisation immédiate sans attendre le cron. |

Un piège de rédaction pour `/briefing` : **une note de recherche n'est pas une tâche.** Il faut recouper avec les mémoires de projet avant d'annoncer des priorités, sinon le briefing invente du travail à faire à partir de notes exploratoires.

---

## 8. Vérification de l'installation

```bash
# 1. Le CLI répond et l'auth tient
notebooklm list --json | jq '.[].title'

# 2. Le vault a la bonne structure
ls "$HOME/Library/Mobile Documents/iCloud~md~obsidian/Documents/Brain Ai"

# 3. Une synchronisation manuelle passe
~/bin/brain-sync.sh && tail -20 ~/.brain-sync.log

# 4. Le fichier d'état s'est bien mis à jour
stat -f "%Sm" ~/.brain-sync-state.json

# 5. L'agent launchd est chargé
launchctl list | grep brain-sync

# 6. Aucun doublon dans le carnet
notebooklm source list --notebook <ID> --json | jq -r '.[].title' | sort | uniq -d
```

La commande 6 doit ne rien retourner. Si elle crache des titres, le fichier d'état ne se met pas à jour — retourner au §4.2.

Test de bout en bout : lancer une courte session Claude Code, faire `/wrapup`, puis vérifier que la note existe dans `Sessions/` **et** que la source apparaît dans le carnet.

---

## 9. Dépannage

| Symptôme | Cause | Correctif |
|---|---|---|
| Doublons massifs dans NotebookLM | Fichier d'état pas persisté | Le sortir d'iCloud (§4.2), dédupliquer avec `source delete-by-title` |
| `launchd` ne fait rien | PATH minimal | Déclarer `EnvironmentVariables` → `PATH` dans le plist |
| Tout échoue d'un coup après des mois | Cookies Google expirés | `notebooklm login` |
| `notebooklm: command not found` dans un script | Le symlink n'est pas dans le PATH de launchd | Chemin absolu `~/.notebooklm-venv/bin/notebooklm` |
| Les notes ne partent pas | Tag `#brain` absent | Vérifier le frontmatter ; l'étiquetage automatique ne couvre que `Projects/`, `Ideas/`, `Notes/` |
| Vue Graphe vide malgré les tags | Pas de wiki links | Les tags ne créent pas de liens — ajouter `**Projets reliés :**` |
| Le vault n'apparaît pas sur iPhone | iCloud pas synchronisé | Vérifier Réglages → iCloud → iCloud Drive → Obsidian |
| Sessions écrites deux fois | `/wrapup` n'a pas mis à jour `pulled_sessions` | Ajouter le titre au tableau à l'étape 4 |

---

## 10. Ordre d'installation

```
1.  Prérequis          brew, jq, python 3.12, ~/bin dans le PATH
2.  Vault Obsidian     structure des dossiers + gabarits
3.  CLI NotebookLM     venv, install, playwright, symlink
4.  notebooklm login   authentification Google
5.  Créer le carnet    NOTER L'UUID
6.  brain-sync.sh      état HORS iCloud, liste d'auto-liens adaptée
7.  Test manuel        lancer le script, lire le journal
8.  launchd            sync aux 2 h + rappel 22 h, avec PATH explicite
9.  Skill /wrapup      avec l'UUID du carnet
10. Test bout en bout  une session → /wrapup → vérifier Obsidian ET NotebookLM
```

Compter environ une heure, dont la moitié à `notebooklm login` et à adapter la liste d'auto-liens.

---

## 11. Ce qui fait vraiment la différence

Le montage technique est la partie facile. Ce qui décide si le système sert à quelque chose dans six mois :

- **Faire le `/wrapup` chaque soir.** C'est la seule discipline requise. Le rappel de 22 h existe pour ça.
- **Toujours tags de projet + wiki links.** Une session sans liens est une session perdue : elle existe mais rien ne la retrouvera.
- **Écrire les mémoires avec le *pourquoi*.** « On a choisi X » ne vaut rien dans trois mois. « On a choisi X parce que Y échouait sur Z » se réutilise.
- **Ne rien mettre de sensible dans le vault.** Tout ce qui porte `#brain` part chez Google. Les clés, mots de passe et données clients restent dehors.
- **Vérifier les doublons une fois par mois** avec la commande 6 du §8. Le bogue du fichier d'état est sournois : il ne casse rien de visible, il gonfle le carnet jusqu'à le rendre inutilisable.
