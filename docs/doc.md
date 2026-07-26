### Structure du projet

```
src/main.ts              → point d'entrée du CLI : arguments (--version, update), chargement .env, boucle interactive
src/planner.ts            → orchestration du planificateur interactif (runCommitPlanner)
src/git.ts                → primitives Git/fichiers (auteur, dates, git init, fichiers non suivis...)
src/claude-sessions.ts     → découverte et parsing des sessions Claude Code (~/.claude/projects)
src/commit-units.ts        → reconstruction chronologique, découpage fin, application réelle des commits
src/claude-cli.ts          → détection et appel en streaming du CLI Claude Code local
src/procedural.ts          → générateur de commits hors-ligne (repli sans IA)
src/self-update.ts         → `cmt update` (téléchargement avec progression, décompression, remplacement du binaire)
src/version.ts             → résolution de la version (`__CMT_VERSION__` injecté au build, ou repli sur package.json en dev)
src/ui/                   → couche terminal : couleurs (colors.ts), prompts/sélecteur à flèches (prompt.ts), bannière (banner.ts), curseur (terminal.ts)
src/diff.d.ts              → déclaration de types locale pour le paquet `diff` (qui n'en fournit pas)
src/globals.d.ts           → déclaration de `__CMT_VERSION__`
src/types.ts               → types hérités de l'ancienne app web compagnon, non utilisés par le CLI
scripts/build-cli.mjs    → bundle src/main.ts avec esbuild, en y injectant la version de package.json
scripts/release.mjs      → bump package.json + commit + tag, en une seule action atomique (npm run release)
installers/install.sh, install.ps1     → installent le binaire compilé sous la commande `cmt`
installers/uninstall.sh, uninstall.ps1 → retirent `cmt`
docs/preview.png         → capture d'écran utilisée dans ce README
.github/workflows/release.yml → compile et publie les 4 binaires sur chaque tag `v*`
```

### Architecture du CLI

Grandes étapes du pipeline, réparties sur plusieurs modules sous `src/` :

1. **Détection des sessions Claude Code** (`src/claude-sessions.ts`) — `locateClaudeCodeDir()`, `encodeProjectPath()` (reproduit l'encodage utilisé par Claude Code pour `~/.claude/projects/<encodé>`), `findProjectSessions()`, `summarizeSession()`, `extractFileChanges()` (parcourt les blocs `tool_use` — `Edit`, `MultiEdit`, `Write`, `NotebookEdit` — des transcripts `.jsonl`).
2. **Reconstruction chronologique** (`src/commit-units.ts`) — `buildCommitUnits()` lit les sauvegardes de versions de fichiers de Claude Code (`~/.claude/file-history/<session>/<hash>@vN`) pour retrouver l'état réel de chaque fichier à chaque étape (à défaut d'historique, diff contre `HEAD`) ; `buildCommitUnitsFromGitDiff()` fait la même chose directement depuis `git status` quand aucune session n'est utilisée.
3. **Découpage fin** (`src/commit-units.ts`) — diff ligne-à-ligne (paquet `diff`) via `expandUnitsToCount()` pour subdiviser une modification en plusieurs commits quand le nombre de "vraies" étapes est insuffisant par rapport au nombre demandé.
4. **Génération des messages** — CLI Claude local en streaming (`src/claude-cli.ts`), puis API Anthropic/Gemini, puis générateur procédural (`src/procedural.ts`), dans cet ordre de priorité — orchestré depuis `src/planner.ts`.
5. **Application réelle** (`src/commit-units.ts`) — `applyCommitUnits()` écrit chaque état historique sur disque, commit, puis restaure garantit l'état réel du fichier (`try/finally`), même en cas d'erreur en cours de route.

### Commandes utiles

```bash
npm run cli        # lance le CLI en mode développement (via tsx, pas de build requis)
npm run lint        # tsc --noEmit — à faire passer avant tout commit
npm run build:cli    # bundle src/main.ts en un seul fichier CJS (dist/cli.cjs)
npm run compile      # build:cli + génère les 4 exécutables autonomes (voir plus haut)
```

### Publier une nouvelle release (binaires `cmt`)

`package.json` est la source de vérité pour la version — elle est embarquée dans le binaire au build (`__CMT_VERSION__`, injecté par `scripts/build-cli.mjs`) et exposée via `cmt --version`. Un tag Git créé séparément d'un bump de `package.json` peut diverger (le tag pointe sur un commit figé, il ne "suit" pas les changements ultérieurs) — `npm run release` fait donc les deux ensemble, en une seule action atomique :

```bash
npm run release -- 0.1.6
# ou un incrément semver : npm run release -- patch / minor / major
```

Ceci bump `package.json` (et `package-lock.json`), commit, et crée le tag `vX.Y.Z` correspondant localement — sans rien pousser. Le script affiche la commande de push à la fin ; l'exécuter déclenche réellement la release :

```bash
git push origin main v0.1.6
```
