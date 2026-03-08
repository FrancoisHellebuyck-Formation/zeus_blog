# CLAUDE.md — Instructions pour Claude

## Contexte du projet

Blog Zola "Zeus Blog". Zeus est un Maine Coon qui veut conquérir l'univers grâce à l'IA agentique — blog de vulgarisation IA/ML avec un ton humoristique.

- **URL** : `https://FrancoisHellebuyck-Formation.github.io/zeus_blog`
- **Auteur** : François H.
- **Stack** : Zola + thème PaperMod, articles dans `content/posts/`, frontmatter TOML
- **Taxonomies** : `tags` uniquement
- **Shortcode Mermaid** : `{% mermaid() %}...{% end %}` (template dans `templates/shortcodes/mermaid.html`, activer via `extra.mermaid = true` dans le frontmatter)
- **Coloration syntaxique** : thème `ayu-dark`

## Ton et style

- Narrateur : Zeus, un Maine Coon conquérant et stratège
- Ton : humoristique, décalé, mais techniquement précis
- Structure type : problème de Zeus → solution technique → code → leçons apprises ("Ce que j'avais mal compris")
- Les posts se terminent par une section "En une phrase" puis une ligne d'accroche italique

## Articles existants

1. **`generation-images-local.md`** (2026-03-07) — API FLUX.2-klein-4B conteneurisée sur alpha-server (port 8000). Génération d'images via Docker + HuggingFace Diffusers, 16 Go VRAM.

2. **`mcp-le-protocole-des-conquerants.md`** (2026-03-08) — Serveur MCP SSE conteneurisé sur alpha-server (port 8080). Trois tools : `generer_image`, `verifier_api_flux`, `fetch_image`. Connexion via `mcp-remote` dans `claude_desktop_config.json`. Diagramme Mermaid de l'architecture.

## Infrastructure

- **alpha-server** : serveur Linux distant accessible via Tailscale (`100.115.15.123`)
  - `zeus-flux` : API FLUX.2-klein-4B conteneurisée, port 8000
  - `zeus-mcp` : serveur MCP SSE conteneurisé, port 8080 — ASGI router custom (pas de Starlette Mount)
- **MCP tools disponibles** : `generer_image`, `verifier_api_flux`, `fetch_image`
- **MCP config** : `mcp-remote` dans `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS)
- **`.mcp.json`** à la racine du projet pour Claude Code CLI

## Workflow nouveau post

1. Lire le skill `zola-new-article` avant de commencer
2. Créer le fichier dans `content/posts/`
3. Générer une image d'illustration avec `generer_image`, la récupérer avec `fetch_image`, la sauvegarder dans `static/images/`
4. Utiliser `{% mermaid() %}` pour les diagrammes d'architecture (+ `extra.mermaid = true` dans le frontmatter)

## Règles importantes

### Fichiers sur alpha-server
Claude n'a pas d'accès réseau direct à alpha-server depuis la VM Cowork.
**Toujours fournir l'intégralité du fichier** quand une modification est nécessaire sur alpha-server (ex: `mcp_server.py`, `docker-compose.yml`, Dockerfile), et non un diff partiel — l'utilisateur doit pouvoir copier-coller directement.

### Git
Ne jamais faire `git push` sauf demande explicite.

### Images
Les images générées par `zeus-image-tools` sont sauvegardées dans le volume Docker `/output` sur alpha-server.
Workflow autonome :
1. Appeler `generer_image` → retourne le filename
2. Appeler `fetch_image` avec le filename → retourne le base64 comme TextContent (sauvegardé dans un fichier temporaire si trop long)
3. Décoder avec Python et sauvegarder dans `static/images/`
