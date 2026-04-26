---
name: help
description: Use when the user asks for help about the vibe-stack plugin, wants to list available skills, or asks "qu'est-ce que ce plugin fait", "vibe-stack help", "/help vibe-stack", or any equivalent. Displays the plugin overview, available skills, upcoming features and useful links.
disable-model-invocation: true
---

# Plugin vibe-stack — Aide

Plugin Claude Code packageant l'écosystème de développement **vibe-stack** : skills, slash commands et conventions partagées pour les projets utilisant la stack Bun + NestJS + React/Vite + tRPC + Prisma + Zod.

## Skills disponibles

- **`vibe-stack:help`** — cette aide
- **`vibe-stack:sync-vibe-stack`** — Synchronise le code template vibe-stack et les modules vers le projet courant. Détecte les releases disponibles, applique les diffs intelligemment et met à jour le `.flow/vibe-stack-lock.json`.

## À venir

Prochaines skills à migrer depuis `~/.claude/skills/` :

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

## Instructions

Affiche directement le contenu ci-dessus à l'utilisateur, formaté en markdown lisible. Ne pas exécuter d'autre action.
