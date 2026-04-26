---
name: hotfix
description: Cherry-pick a commit to production branch and deploy
argument-hint: "[commit-hash]"
disable-model-invocation: true
---

# Hotfix

Cherry-pick un commit spécifique vers la branche de production sans merger toute la branche de développement.

## Step 0: Detect branches

1. Chercher `deploy.json` à la racine du projet
2. **Si `deploy.json` existe avec `environments`** :
   - Extraire la branche prod : celle où `env_name` contient "prod" ou est la dernière listée
   - Extraire la branche dev : celle où `env_name` contient "test"/"dev"/"staging" ou est la première listée
   - Afficher : "Branches détectées : dev=`{dev_branch}`, prod=`{prod_branch}`"
3. **Sinon, détecter par convention** :
   - Prod : vérifier si `main` existe, sinon `master`
   - Dev : vérifier si `develop` existe, sinon `dev`, sinon la branche courante
   - Proposer de créer `deploy.json` avec ces valeurs via `AskUserQuestion`
4. Vérifier qu'on n'est PAS déjà sur la branche prod

## Step 1: Select commit to cherry-pick

1. Si un argument `commit-hash` est fourni, l'utiliser
2. Sinon, afficher les 10 derniers commits de la branche courante :
   ```bash
   git log --oneline -10
   ```
3. Demander via `AskUserQuestion` : "Quel commit cherry-pick vers prod ?"
   - Proposer les 3-4 derniers commits comme options
   - Option "Autre" pour saisir un hash manuellement

## Step 2: Cherry-pick to prod

1. Vérifier que le working directory est propre (`git status`)
2. Fetch et checkout la branche prod :
   ```bash
   git fetch origin
   git checkout {prod_branch}
   git pull origin {prod_branch}
   ```
3. Cherry-pick le commit :
   ```bash
   git cherry-pick {commit_hash}
   ```
4. **Si conflit** :
   - Afficher les fichiers en conflit
   - Demander : "Résoudre manuellement et continuer, ou annuler ?"
   - Si annuler : `git cherry-pick --abort` et revenir à la branche d'origine
5. **Si succès** :
   - Afficher le nouveau commit créé
   - Demander : "Pusher ce commit vers prod ?"

## Step 3: Push and deploy

1. Si l'utilisateur confirme :
   ```bash
   git push origin {prod_branch}
   ```
2. Surveiller le workflow GitHub Actions (comme dans `/deploy`)
3. Vérifier le health check du serveur prod

## Step 4: Return to original branch

1. Revenir à la branche d'origine :
   ```bash
   git checkout {original_branch}
   ```
2. Afficher un résumé :
   ```
   Hotfix Summary
   --------------
   Commit:      {commit_hash} - {commit_message}
   Applied to:  {prod_branch}
   Workflow:    {passed/failed}
   Health:      {OK/FAIL}
   ```

## Important notes

- TOUJOURS vérifier que le working directory est propre avant de commencer
- TOUJOURS revenir à la branche d'origine à la fin (même en cas d'erreur)
- Ne JAMAIS forcer le push (`--force`)
- Si le cherry-pick échoue, proposer d'annuler proprement
