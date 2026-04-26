---
name: release
description: Show diff between dev and prod branches, merge to prod and deploy
disable-model-invocation: true
---

# Release

Affiche le delta entre les branches dev et prod, propose de merger et déployer en production.

## Step 0: Detect branches

1. Chercher `deploy.json` à la racine du projet
2. **Si `deploy.json` existe avec `environments`** :
   - Extraire la branche prod : celle où `env_name` contient "prod" ou est la dernière listée
   - Extraire la branche dev : celle où `env_name` contient "test"/"dev"/"staging" ou est la première listée
   - Afficher : "Branches détectées : dev=`{dev_branch}`, prod=`{prod_branch}`"
3. **Sinon, détecter par convention** :
   - Prod : vérifier si `main` existe, sinon `master`
   - Dev : vérifier si `develop` existe, sinon `dev`
   - Proposer de créer `deploy.json` avec ces valeurs via `AskUserQuestion`

## Step 1: Show delta

1. Fetch les dernières modifications :
   ```bash
   git fetch origin
   ```
2. Afficher le nombre de commits en avance :
   ```bash
   git rev-list --count origin/{prod_branch}..origin/{dev_branch}
   ```
3. Afficher la liste des commits :
   ```bash
   git log origin/{prod_branch}..origin/{dev_branch} --oneline
   ```
4. Afficher un résumé des fichiers modifiés :
   ```bash
   git diff origin/{prod_branch}..origin/{dev_branch} --stat
   ```
5. Présenter un résumé clair :
   ```
   Release Summary
   ---------------
   From:     {dev_branch}
   To:       {prod_branch}
   Commits:  {N} commits
   Files:    {M} files changed

   Commits to merge:
   - abc1234 feat(xxx): ...
   - def5678 fix(yyy): ...
   ```

## Step 2: Confirm merge

1. Demander via `AskUserQuestion` :
   - "Merger ces {N} commits vers prod ?"
   - Options : "Oui, merger et déployer", "Non, annuler"
2. Si "Non" : s'arrêter et revenir à la branche d'origine

## Step 3: Merge to prod

1. Vérifier que le working directory est propre
2. Checkout et pull la branche prod :
   ```bash
   git checkout {prod_branch}
   git pull origin {prod_branch}
   ```
3. Merger la branche dev :
   ```bash
   git merge origin/{dev_branch} --no-edit
   ```
4. **Si conflit** :
   - Afficher les fichiers en conflit
   - Demander : "Résoudre manuellement, ou annuler ?"
   - Si annuler : `git merge --abort` et revenir à la branche d'origine
5. **Si succès** :
   - Push vers prod :
     ```bash
     git push origin {prod_branch}
     ```

## Step 4: Monitor deployment

1. Surveiller le workflow GitHub Actions :
   ```bash
   gh run list --workflow=<deploy-workflow> --branch={prod_branch} --limit=1
   gh run watch <run-id>
   ```
2. Si le workflow réussit, vérifier le serveur prod (via `deploy.json` si disponible) :
   ```bash
   curl -sf {health_check_url}
   ```

## Step 5: Return and summary

1. Revenir à la branche d'origine :
   ```bash
   git checkout {original_branch}
   ```
2. Afficher le résumé final :
   ```
   Release Complete
   ----------------
   Merged:      {dev_branch} → {prod_branch}
   Commits:     {N} commits
   Workflow:    {passed/failed}
   Health:      {OK/FAIL}
   Production:  {health_check_url}
   ```

## Important notes

- TOUJOURS afficher le delta avant de proposer le merge
- TOUJOURS vérifier que dev est à jour avec origin avant de merger
- TOUJOURS revenir à la branche d'origine à la fin
- Ne JAMAIS forcer le push (`--force`)
- Si le merge échoue, proposer d'annuler proprement
