+++
title = "MCP — donner à Claude la commande de l'armurerie de Zeus"
date = 2026-03-08
description = "L'API FLUX tourne sur alpha-server. Bien. Maintenant Zeus veut que Claude Cowork génère des images de façon autonome, sans curl manuel. La solution : un serveur MCP conteneurisé sur alpha-server, connecté via SSE."
draft = false

[extra]
mermaid = true

[taxonomies]
tags = ["mcp", "agents", "protocole", "python", "llm", "image-generation", "flux", "docker", "sse"]
+++

![Zeus supervise son armurerie depuis le centre de commandement](/images/zeus_mcp_command_center.png)

[Dans l'épisode précédent](/posts/generation-images-local/), Zeus avait résolu son problème de visuels : une API FLUX.2-klein-4B conteneurisée sous Docker, qui tourne sur **alpha-server** et génère des portraits de l'armée féline en 20 à 40 secondes.

Zeus a maintenant un problème différent.

L'API est là. Elle fonctionne. Mais à chaque fois qu'un agent veut une image, *quelqu'un* doit écrire le `curl`, passer le prompt, décoder le base64, sauvegarder le fichier. Ce quelqu'un, c'est Zeus. Zeus n'a pas que ça à faire. Zeus dirige une invasion galactique.

Il faut que **Claude Cowork génère les images lui-même**, de façon autonome, quand il en a besoin. On lui tend un outil. Claude l'utilise. Zeus supervise depuis son trône.

C'est exactement ce que permet **MCP — Model Context Protocol**.

---

## L'architecture cible

Tout tourne sur **alpha-server**. Le poste local fait tourner `mcp-remote` en stdio — un pont léger qui traduit les appels de Claude Desktop en SSE HTTP vers alpha-server via Tailscale.

{% mermaid() %}
graph TB
    subgraph local["Poste local — macOS"]
        A["Claude Desktop / Cowork"]
        P["mcp-remote<br/>(processus stdio local)<br/>npx mcp-remote --allow-http"]
    end

    subgraph tailscale["Tailscale — WireGuard chiffré"]
        T(["100.115.15.123"])
    end

    subgraph alpha["alpha-server — Docker network : zeus-net"]
        B["zeus-mcp<br/>port 8080<br/>ASGI custom — SSE"]
        C["zeus-flux<br/>port 8000<br/>API FLUX.2-klein-4B"]
    end

    A -->|"stdio MCP"| P
    P -->|"GET /sse/ — SSE stream"| T
    P -->|"POST /messages/ — tools calls"| T
    T --> B
    B -->|"HTTP POST /generate"| C
    C -->|"image PNG base64"| B
    B -->|"TextContent + ImageContent"| P
    P -->|"image inline"| A
{% end %}

Deux conteneurs sur le même réseau Docker interne `zeus-net`. Le pont `mcp-remote` tourne localement en stdio — il traduit le protocole stdio de Claude Desktop en SSE HTTP vers alpha-server via Tailscale. Le flag `--allow-http` lève la restriction HTTPS, le chiffrement WireGuard de Tailscale assurant la sécurité du transit.

---

## Pourquoi SSE et pas stdio ?

En mode **stdio**, Claude Desktop *lance lui-même* le processus serveur en local via `command` + `args`. Impossible sur un serveur distant.

En mode **SSE** (Server-Sent Events), le serveur MCP est un service HTTP persistant. Claude Desktop s'y connecte via une URL. C'est le mode prévu pour les serveurs distants — et aussi pour les cas où plusieurs clients veulent se connecter au même serveur.

---

## Le serveur MCP — `mcp_server.py`

```python
import base64
import datetime
import pathlib
import os

import requests
from mcp import types
from mcp.server import Server
from mcp.server.sse import SseServerTransport
from starlette.responses import Response
import uvicorn

# L'API FLUX est accessible via le réseau Docker interne
FLUX_API_URL = os.getenv("FLUX_API_URL", "http://zeus-flux:8000/generate")
FLUX_HEALTH_URL = os.getenv("FLUX_HEALTH_URL", "http://zeus-flux:8000/health")

# Les images sont sauvegardées dans un dossier monté en volume
OUTPUT_DIR = pathlib.Path(os.getenv("OUTPUT_DIR", "/output"))
OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

server = Server("zeus-image-tools")


@server.list_tools()
async def list_tools() -> list[types.Tool]:
    return [
        types.Tool(
            name="generer_image",
            description=(
                "Génère une image à partir d'un prompt textuel en utilisant FLUX.2-klein-4B. "
                "Retourne le chemin vers l'image PNG sauvegardée. "
                "Utilise cet outil dès qu'on te demande de créer, générer ou illustrer quelque chose."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "prompt": {
                        "type": "string",
                        "description": "Description en anglais de l'image à générer.",
                    },
                    "width": {
                        "type": "integer",
                        "description": "Largeur en pixels (défaut : 512).",
                        "default": 512,
                    },
                    "height": {
                        "type": "integer",
                        "description": "Hauteur en pixels (défaut : 512).",
                        "default": 512,
                    },
                    "steps": {
                        "type": "integer",
                        "description": "Nombre d'étapes de diffusion (défaut : 4).",
                        "default": 4,
                    },
                },
                "required": ["prompt"],
            },
        ),
        types.Tool(
            name="verifier_api_flux",
            description="Vérifie que l'API FLUX est en ligne et prête à générer des images.",
            inputSchema={"type": "object", "properties": {}},
        ),
        types.Tool(
            name="fetch_image",
            description="Retourne le contenu base64 d'une image sauvegardée dans /output/. Permet à Claude de récupérer l'image et de la sauvegarder localement.",
            inputSchema={
                "type": "object",
                "properties": {
                    "filename": {
                        "type": "string",
                        "description": "Nom du fichier dans /output/.",
                    },
                },
                "required": ["filename"],
            },
        ),
    ]


@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[types.TextContent]:

    if name == "verifier_api_flux":
        try:
            r = requests.get(FLUX_HEALTH_URL, timeout=5)
            data = r.json()
            return [types.TextContent(
                type="text",
                text=f"API en ligne. Modèle : {data.get('model')}. Status : {data.get('status')}.",
            )]
        except Exception as e:
            return [types.TextContent(type="text", text=f"API hors ligne : {e}")]

    if name == "fetch_image":
        filepath = OUTPUT_DIR / arguments["filename"]
        try:
            with open(filepath, "rb") as f:
                b64 = base64.b64encode(f.read()).decode()
            return [types.TextContent(type="text", text=b64)]
        except FileNotFoundError:
            return [types.TextContent(type="text", text=f"Fichier introuvable : {filepath}")]

    if name == "generer_image":
        prompt = arguments["prompt"]
        width = arguments.get("width", 512)
        height = arguments.get("height", 512)
        steps = arguments.get("steps", 4)

        try:
            response = requests.post(
                FLUX_API_URL,
                json={
                    "prompt": prompt,
                    "width": width,
                    "height": height,
                    "steps": steps,
                    "guidance_scale": 0.0,
                },
                timeout=120,
            )
            response.raise_for_status()
        except requests.exceptions.ConnectionError:
            return [types.TextContent(
                type="text",
                text="Erreur : impossible de joindre zeus-flux:8000. Le conteneur FLUX est-il démarré ?",
            )]
        except Exception as e:
            return [types.TextContent(type="text", text=f"Erreur API FLUX : {e}")]

        image_b64 = response.json()["image_base64"]
        timestamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
        slug = "".join(c if c.isalnum() else "_" for c in prompt[:40]).strip("_")
        filename = f"{timestamp}_{slug}.png"
        filepath = OUTPUT_DIR / filename

        with open(filepath, "wb") as f:
            f.write(base64.b64decode(image_b64))

        return [
            types.TextContent(
                type="text",
                text=f"Image générée : {filepath}\nPrompt : {prompt}\nDimensions : {width}×{height}",
            ),
            types.ImageContent(
                type="image",
                data=image_b64,
                mimeType="image/png",
            ),
        ]

    raise ValueError(f"Outil inconnu : {name}")


# ── ASGI router custom — pas de path stripping ─────────────────────────────────
#
# Pourquoi pas Starlette Router ?
# Mount("/sse", ...) strip le path → handle_post_message reçoit "/" au lieu de
# "/messages/" → ne trouve pas la session → HTTP 405. On route manuellement
# pour transmettre le scope intact aux deux handlers.

sse_transport = SseServerTransport("/messages/")


async def app(scope, receive, send):
    if scope["type"] == "lifespan":
        return
    path = scope.get("path", "")
    method = scope.get("method", "GET").upper()

    if path.rstrip("/") == "/sse":
        # Rejeter les POST : mcp-remote tente d'abord Streamable HTTP (POST),
        # le 405 le force à basculer vers le fallback SSE classique (GET).
        if method != "GET":
            await Response(status_code=405)(scope, receive, send)
            return
        async with sse_transport.connect_sse(scope, receive, send) as streams:
            await server.run(streams[0], streams[1], server.create_initialization_options())

    elif path.startswith("/messages"):
        await sse_transport.handle_post_message(scope, receive, send)

    else:
        await Response(status_code=404)(scope, receive, send)


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8080)
```

Trois tools au total : `generer_image` appelle FLUX et retourne l'image inline via `ImageContent` ; `verifier_api_flux` vérifie que l'API est disponible ; `fetch_image` retourne le base64 d'une image existante comme `TextContent` — ce qui permet à Claude de récupérer une image générée précédemment et de la sauvegarder directement dans le dossier du blog, sans `scp`.

La différence clé avec un serveur stdio : on ne lance plus `stdio_server()` mais une **ASGI app custom** avec deux routes — `/sse` pour ouvrir la connexion SSE, `/messages` pour recevoir les appels de tools. On évite volontairement `Starlette.Mount` qui strip les paths et casse le lookup de session dans `handle_post_message`. Uvicorn sert le tout sur le port 8080.

---

## Le Dockerfile du serveur MCP

```dockerfile
FROM python:3.12-slim

WORKDIR /app

RUN pip install --no-cache-dir \
    "mcp==1.4.1" \
    "requests==2.32.3" \
    "starlette==0.46.1" \
    "uvicorn==0.34.0"

COPY mcp_server.py .

EXPOSE 8080

CMD ["python", "mcp_server.py"]
```

Léger : pas de GPU, pas de PyTorch. Ce conteneur ne fait que router des appels HTTP.

---

## docker-compose.yml — les deux services ensemble

On reprend le `docker-compose.yml` de l'article précédent et on ajoute le service MCP. Les deux partagent le réseau interne `zeus-net` et un volume pour les images générées.

```yaml
services:

  zeus-flux:
    build:
      context: ./flux          # le Dockerfile de l'article précédent
    ports:
      - "8000:8000"
    volumes:
      - ~/.cache/huggingface:/root/.cache/huggingface
    environment:
      - HF_TOKEN=${HF_TOKEN}
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    networks:
      - zeus-net
    restart: unless-stopped

  zeus-mcp:
    build:
      context: ./mcp           # le Dockerfile ci-dessus
    ports:
      - "8080:8080"            # exposé vers le poste local
    volumes:
      - zeus-images:/output    # images partagées entre les deux conteneurs
    environment:
      - FLUX_API_URL=http://zeus-flux:8000/generate
      - FLUX_HEALTH_URL=http://zeus-flux:8000/health
      - OUTPUT_DIR=/output
    networks:
      - zeus-net
    depends_on:
      - zeus-flux
    restart: unless-stopped

networks:
  zeus-net:

volumes:
  zeus-images:
```

Structure du projet sur alpha-server :

```
zeus-server/
├── docker-compose.yml
├── flux/
│   ├── Dockerfile       ← article précédent
│   └── app.py
└── mcp/
    ├── Dockerfile
    └── mcp_server.py
```

Lancement :

```bash
docker compose up --build -d
```

---

## Configuration de Cowork

Plutôt que de modifier la config globale de Claude Desktop, on peut déclarer le serveur MCP **localement dans le projet** via un fichier `.mcp.json` à la racine. La config voyage avec le repo.

```json
{
  "mcpServers": {
    "zeus-image-tools": {
      "url": "http://alpha-server:8080/sse"
    }
  }
}
```

Remplace `alpha-server` par l'IP ou le hostname réel. Cowork (basé sur Claude Code) détecte automatiquement ce fichier au démarrage d'une session dans le dossier du projet — les outils apparaissent sans toucher à la configuration système.

### Claude Desktop refuse le HTTP ? La solution `mcp-remote`

Petit piège découvert en lab : **Claude Desktop et Cowork refusent les connexions MCP en HTTP**. Le `.mcp.json` avec un `url` en `http://` est silencieusement ignoré. Le protocole exige HTTPS.

Mettre en place TLS sur un serveur local ou Tailscale demande du travail (cert auto-signé, reverse proxy, confiance système). Il existe une solution plus directe : **`mcp-remote`**, un pont stdio↔SSE qui s'exécute localement.

Le principe : Claude Desktop lance `mcp-remote` comme un processus stdio local (ce qu'il sait parfaitement faire), et `mcp-remote` se charge lui-même de se connecter au serveur SSE distant. Le flag `--allow-http` lève la restriction HTTPS côté pont.

Dans `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) :

```json
{
  "mcpServers": {
    "zeus-image-tools": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "http://100.115.15.123:8080/sse/",
        "--allow-http"
      ]
    }
  }
}
```

Aucune installation préalable — `npx -y` télécharge et exécute `mcp-remote` à la demande. Remplace `100.115.15.123` par l'IP Tailscale de ton serveur.

Redémarre Claude Desktop après modification. Les outils `zeus-image-tools` apparaissent dans Cowork comme si de rien n'était — Zeus n'a jamais su qu'il y avait eu un problème.

> **Note** : cette config va dans `claude_desktop_config.json` (globale, tous projets), pas dans `.mcp.json` (locale, projet unique). Les deux coexistent sans conflit.

> **Sécurité** : si alpha-server est accessible depuis Internet, ajoute un reverse proxy (nginx, Caddy) avec authentification devant le port 8080. En réseau local fermé ou sur Tailscale, HTTP suffit — le trafic est déjà chiffré par WireGuard.

---

## Ce que j'avais mal compris

**"stdio fonctionne à distance"** — Non. stdio ne fonctionne qu'en local : Claude Desktop lance le processus serveur lui-même sur la même machine. Dès que le serveur est sur une autre machine, il faut SSE (ou un autre transport HTTP). C'est la distinction fondamentale entre les deux modes.

**"localhost dans le conteneur pointe vers l'API FLUX"** — Non. Dans un conteneur Docker, `localhost` pointe vers le conteneur lui-même. Pour joindre un autre conteneur sur le même réseau Docker, on utilise son **nom de service** — ici `zeus-flux:8000`. Docker résout le nom via son DNS interne.

**"Le serveur MCP a besoin de GPU"** — Non. Le serveur MCP ne fait que router des requêtes HTTP. Tout le travail GPU est dans le conteneur `zeus-flux`. Le conteneur `zeus-mcp` tourne sur une image Python slim sans aucune dépendance CUDA.

**"`pip install` sans versions épinglées est stable"** — Non. Un `docker compose up --build` sans versions figées tire les dernières releases. Entre deux builds, une nouvelle version de Starlette peut casser le handler SSE : `Route(endpoint=func)` attend un `Response` en retour, ce que le handler SSE ne fait pas. Fix : passer en ASGI pur avec une fonction `(scope, receive, send)` — plus de dépendance à `request._send`. Et épingler les versions dans le Dockerfile.

**"`Mount` de Starlette ne casse pas le routing SSE"** — Si. `Mount("/sse", app=handler)` strip le path : le handler reçoit `/` au lieu de `/sse/`, et `Mount("/messages", ...)` strip aussi — `handle_post_message` reçoit `/` au lieu de `/messages/?session_id=xxx`, ne trouve pas la session, retourne HTTP 405. Fix : router manuellement avec un ASGI app custom qui transmet le `scope` intact aux deux handlers.

**"Renvoyer le chemin du fichier suffit"** — Non. Renvoyer un chemin texte oblige à faire un `scp` pour récupérer l'image. Le SDK MCP supporte nativement `ImageContent` — en renvoyant `data` (base64) et `mimeType`, Claude affiche directement l'image dans la conversation. `image_b64` est déjà en mémoire au moment du `return`, pas besoin de relire le fichier.

**"Le `.mcp.json` avec `http://` fonctionne dans Cowork"** — Non. Claude Desktop refuse les connexions MCP en HTTP, même en réseau local. Le `.mcp.json` avec une URL `http://` est ignoré sans message d'erreur explicite. La solution : utiliser `mcp-remote` avec `--allow-http` dans `claude_desktop_config.json`, qui fait tourner un pont stdio local vers le serveur SSE distant.

**"Il faut redémarrer Claude Desktop à chaque changement"** — Seulement si on modifie le fichier `claude_desktop_config.json`. Le code du serveur MCP, lui, peut être redéployé avec `docker compose up --build -d zeus-mcp` sans toucher à la config locale.

---

## En une phrase

Un conteneur MCP de 40 lignes utiles suffit à exposer l'API FLUX d'alpha-server comme outil natif dans Claude Cowork — Claude génère les images de façon autonome via SSE, les sauvegarde dans un volume partagé, et Zeus n'a plus à intervenir entre le plan de conquête et les visuels.

---

*L'armurerie est sur alpha-server. Claude a les clés. Zeus peut enfin se concentrer sur la stratégie.*
