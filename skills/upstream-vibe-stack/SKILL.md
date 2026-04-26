---
name: upstream-vibe-stack
description: Use when the user wants to push an improvement from the current project back to vibe-stack template or vibe-stack-modules.
disable-model-invocation: true
---

# Upstream Vibe Stack

Remonter une amelioration du projet courant vers vibe-stack (core/conventions) ou vibe-stack-modules (module).

Ce skill est UPSTREAM uniquement : du projet vers les sources.

## Step 1 — Identifier ce qu'on remonte

Analyser `$ARGUMENTS` s'il y en a. Sinon, demander a l'utilisateur :

"Qu'est-ce que tu veux remonter vers vibe-stack ?"
- Un ou plusieurs commits recents (ex: "les 3 derniers commits")
- Un fichier ou dossier specifique (ex: "apps/api/src/modules/auth/")
- Une description libre (ex: "le nouveau pattern de validation")

## Step 2 — Analyser le scope

Selon la reponse :
- Si commit(s) : `git show <commit>` ou `git diff <commit1>..<commitN>` pour voir les changements
- Si fichier/dossier : lire les fichiers, comprendre ce qui a change par rapport a la version de base (utiliser le lock si disponible)
- Si description : chercher dans le code et les commits recents ce qui correspond

## Step 3 — Verifier la synchronisation

**AVANT** de continuer, verifier que le projet est a jour avec le repo cible.

### Si destination = vibe-stack (core)

```bash
cd ~/Dev/vibe-stack && git pull
LOCK_COMMIT=$(jq -r '.core.commit' .flow/vibe-stack-lock.json)
UNSYNC=$(git log --oneline $LOCK_COMMIT..HEAD)
```

### Si destination = vibe-stack-modules

```bash
cd ~/Dev/vibe-stack-modules && git pull
LOCK_COMMIT=$(jq -r '.modules.<name>.commit' .flow/vibe-stack-lock.json)
UNSYNC=$(git log --oneline $LOCK_COMMIT..HEAD)
```

### Si des commits non synchronises existent

**BLOQUER l'upstream** et afficher :

> ⛔ Le projet n'est pas a jour avec [vibe-stack|vibe-stack-modules].
> Il y a N commit(s) non synchronise(s) :
>
> - `abc1234` description du commit
> - `def5678` description du commit
>
> Utilise `/sync-vibe-stack` d'abord pour tirer ces changements, puis relance `/upstream-vibe-stack`.

Ne pas continuer. L'utilisateur doit d'abord synchroniser pour eviter de perdre des changements ou d'avoir un lock incoherent.

### Si le projet est a jour

Continuer normalement.

## Step 4 — Determiner la destination

Analyser la nature du changement :

| Nature | Destination | Action |
|--------|-------------|--------|
| Code d'un module (service, component, hook) | `~/Dev/vibe-stack-modules/modules/<name>/` | Mettre a jour le module |
| Boilerplate, config, structure de projet | `~/Dev/vibe-stack/` | Mettre a jour le template |
| Conventions, regles, patterns documentes | `~/.claude/shared/vibe-stack.md` | Mettre a jour les conventions |

Si ambigu, demander a l'utilisateur.

Verifier `.flow/vibe-stack-lock.json` pour identifier si les fichiers modifies font partie d'un module connu (via le mapping `files`).

## Step 5 — Presenter le plan

Montrer a l'utilisateur :
```
Amelioration detectee : [description]

Source : [fichiers/commits dans le projet]
Destination : [vibe-stack | vibe-stack-modules/modules/<name> | ~/.claude/shared/vibe-stack.md]

Changements a appliquer :
  - [resume des modifications]

Generalisation necessaire :
  - [ce qu'il faut abstraire pour rendre le code reutilisable]
  - ou "Aucune — le code est deja generique"
```

Demander confirmation avant de continuer.

## Step 6 — Generaliser si necessaire

Si le code contient des specificites du projet :
- Remplacer les references specifiques au projet par des patterns generiques
- Abstraire les enums/types specifiques
- Retirer les imports de services specifiques au projet
- Suivre les conventions de `~/Dev/vibe-stack-modules/CONTRIBUTING.md` si c'est un module

Si c'est une convention/regle : formuler de maniere generique pour vibe-stack.md

## Step 7 — Appliquer dans le repo cible

### Si destination = vibe-stack-modules

1. Modifier les fichiers dans `~/Dev/vibe-stack-modules/modules/<name>/`
2. Mettre a jour le README du module si l'API a change
3. Commiter et pusher :
   ```
   cd ~/Dev/vibe-stack-modules && git add modules/<name>/ && git commit && git push
   ```

### Si destination = vibe-stack

1. Modifier les fichiers dans `~/Dev/vibe-stack/`
2. Commiter et pusher :
   ```
   cd ~/Dev/vibe-stack && git add . && git commit && git push
   ```

### Si destination = vibe-stack.md

1. Modifier `~/.claude/shared/vibe-stack.md`
2. Pas de git (c'est un fichier local partage)

## Step 8 — Mettre a jour le lock du projet

Apres l'upstream, le lock **DOIT** etre mis a jour vers le commit qu'on vient de creer. Puisque le step 3 a verifie qu'on etait a jour avant l'upstream, et que le code pousse vient de notre projet, le lock est forcement coherent.

### Si destination = vibe-stack (core)

Mettre a jour `core.commit` et `core.date` dans `.flow/vibe-stack-lock.json` vers le commit qui vient d'etre cree.

### Si destination = vibe-stack-modules

Mettre a jour le `commit` et la `date` du module dans `.flow/vibe-stack-lock.json` vers le commit qui vient d'etre cree. Le mapping `files` reste inchange (sauf si de nouveaux fichiers ont ete ajoutes au module, auquel cas les ajouter).

### Si destination = vibe-stack.md

Pas de lock a mettre a jour (fichier local).

### Commit du lock

Commiter le lock mis a jour dans le projet courant.

## Step 9 — Resume

```
Amelioration remontee vers [destination]

Commit dans [repo cible] : <hash>
Lock mis a jour dans le projet : <hash>

[Si vibe-stack ou vibe-stack-modules] Les autres projets pourront
recuperer cette amelioration via /sync-vibe-stack
```

## Repos de reference

- Template vibe-stack : `~/Dev/vibe-stack/`
- Modules : `~/Dev/vibe-stack-modules/`
- Conventions partagees : `~/.claude/shared/vibe-stack.md`
- Lock file : `.flow/vibe-stack-lock.json` (dans le projet courant)
