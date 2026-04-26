---
description: Affiche l'aide du plugin vibe-stack (skills, commandes, liens utiles).
---

# Plugin vibe-stack — Aide

Plugin Claude Code packageant l'écosystème de développement **vibe-stack** : skills, slash commands et conventions partagées pour les projets utilisant la stack Bun + NestJS + React/Vite + tRPC + Prisma + Zod.

## Slash commands disponibles

- `/vibe-stack:help` — cette aide

## Skills disponibles

Les skills sont invoquées automatiquement par Claude quand le contexte le justifie, ou explicitement via `/skill vibe-stack:<nom-skill>`.

- **`sync-vibe-stack`** — Synchronise le code template vibe-stack et les modules vers le projet courant. Détecte les releases disponibles, applique les diffs intelligemment et met à jour le `.flow/vibe-stack-lock.json`.

## À venir

Prochaines skills à migrer :

- `import-module` / `export-module` — Gestion des modules vibe-stack
- `deploy` / `deploy-setup` — Déploiement de projets vibe-stack
- `release` / `release-vibe-stack` — Releases du template vibe-stack
- `init-project` — Bootstrap d'un nouveau projet vibe-stack
- `hotfix` — Workflow hotfix
- `commit` — Commits structurés

## Liens

- Repo plugin : https://github.com/vibe-flow/vibe-stack-plugin
- Repo template : https://github.com/vibe-flow/vibe-stack
- Repo modules : https://github.com/vibe-flow/vibe-stack-modules
