+++
title = "Générer des images en local avec FLUX.2-klein — le guide de Zeus"
date = 2026-03-07
description = "Zeus a besoin de portraits de son armée féline. On conteneurise FLUX.2-klein-4B avec Docker et HuggingFace Diffusers pour une API locale de génération d'images qui tient sur 16 Go de VRAM."
draft = false

[taxonomies]
tags = ["image-generation", "flux", "diffusers", "local", "diffusion", "quantization", "docker"]
+++

{{ img(path="images/zeus_portrait.png", alt="Zeus, Maine Coon en quête de domination galactique") }}

Zeus a un problème.

Il a un plan de conquête galactique, un Collectif Félin Mondial motivé, et zéro visuel pour ses slides de présentation. Les chats ne savent pas tenir un appareil photo. Il lui faut un générateur d'images. En local. Dans un conteneur. Parce que Zeus ne fait confiance à aucun cloud, et ne veut pas non plus que ses dépendances Python souillent son système hôte.

Voilà pourquoi on conteneurise **FLUX.2-klein-4B** avec **Docker + HuggingFace Diffusers**.

---

## Pourquoi FLUX.2-klein-4B ?

FLUX.2-klein existe en deux tailles : **4B** et **9B**. Le 9B demande 20+ Go de VRAM — hors budget pour une RTX 4060 Ti 16 Go. Le **4B tourne à partir de 12 Go** avec le CPU offloading, et reste dans les 16 Go disponibles en bfloat16.

Deux autres avantages décisifs par rapport à FLUX.1-schnell :

- **Licence Apache 2.0** — pas de dépôt gated, pas d'acceptation de licence, pas de token HuggingFace obligatoire pour télécharger
- **text-to-image ET image-editing** — Zeus peut générer des portraits *et* les retoucher dans le même pipeline

> Zeus ne choisit pas le meilleur outil sur le papier. Il choisit celui qui tourne vraiment sur sa machine.

---

## Pourquoi pas vLLM ?

La vraie réponse mérite d'être dite clairement : **vLLM ne supporte pas les modèles de diffusion pour la génération d'images** — ni en version standard, ni dans vLLM-Omni qui est encore trop jeune (en tout cas en mars 2026). Zeus a vérifié. Zeus a tenté. Zeus a regardé les issues GitHub avec ses yeux jaunes perçants.

vLLM est excellent pour ce pourquoi il a été conçu : servir des **LLM et des modèles vision-langage** en production. Pour la génération d'images, l'écosystème de référence reste **HuggingFace Diffusers** — et Docker assure l'isolation et la reproductibilité.

---

## Prérequis

- **Docker** + [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- Une GPU NVIDIA avec **au moins 12 Go de VRAM** (16 Go recommandé)
- ~15 Go d'espace disque pour les poids du modèle
- ~8 Go pour l'image Docker de base

Zeus tourne sur une **RTX 4060 Ti 16 Go** avec CUDA 12.9. Vérifie que Docker voit bien ta GPU :

```bash
docker run --rm --gpus all pytorch/pytorch:2.6.0-cuda12.6-cudnn9-devel nvidia-smi
```

Si tu vois le tableau avec ta carte et ta version CUDA, c'est bon. Zeus approuve.

---

## Structure du projet

```
flux-zeus/
├── Dockerfile
├── app.py
└── docker-compose.yml   # optionnel
```

---

## Le serveur FastAPI — `app.py`

```python
import torch
import base64
import io
from fastapi import FastAPI
from pydantic import BaseModel
from diffusers import Flux2KleinPipeline

app = FastAPI(title="Zeus Image API")

print("Chargement de FLUX.2-klein-4B...")

pipe = Flux2KleinPipeline.from_pretrained(
    "black-forest-labs/FLUX.2-klein-4B",
    torch_dtype=torch.bfloat16,
)

# CPU offloading — déplace les composants inutilisés sur le CPU pendant l'inférence
# Nécessaire sur 16 Go : les activations du VAE decoder dépassent la VRAM disponible
pipe.enable_model_cpu_offload()

print("Modèle prêt. Zeus peut générer.")


class ImageRequest(BaseModel):
    prompt: str
    steps: int = 4
    width: int = 512
    height: int = 512
    guidance_scale: float = 0.0


@app.post("/generate")
def generate(req: ImageRequest):
    image = pipe(
        prompt=req.prompt,
        num_inference_steps=req.steps,
        guidance_scale=req.guidance_scale,
        width=req.width,
        height=req.height,
    ).images[0]

    buf = io.BytesIO()
    image.save(buf, format="PNG")
    encoded = base64.b64encode(buf.getvalue()).decode()
    return {"image_base64": encoded}


@app.get("/health")
def health():
    return {"status": "ok", "model": "FLUX.2-klein-4B"}
```

---

## Le Dockerfile

```dockerfile
# Image PyTorch officielle — CUDA + nvcc + headers déjà configurés
FROM pytorch/pytorch:2.6.0-cuda12.6-cudnn9-devel

ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN pip install --no-cache-dir \
    diffusers \
    transformers \
    accelerate \
    fastapi \
    uvicorn \
    pydantic

COPY app.py .

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Build et lancement

### 1. Build de l'image

```bash
docker build -t zeus-flux .
```

### 2. Lancement

```bash
docker run --gpus all \
  -p 8000:8000 \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  zeus-flux
```

Le `-v ~/.cache/huggingface:/root/.cache/huggingface` monte le cache HuggingFace dans le conteneur — les ~15 Go de poids ne se téléchargent qu'une seule fois, même si tu relances ou rebuildes l'image.

### Optionnel — docker-compose.yml

```yaml
services:
  zeus-flux:
    build: .
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
    restart: unless-stopped
```

Le token est lu depuis ton shell via `${HF_TOKEN}` — pas de valeur en dur dans le fichier. Crée un `.env` à côté du `docker-compose.yml` si tu veux le fixer :

```bash
# .env  — ne jamais committer ce fichier
HF_TOKEN=hf_xxxxxxxxxxxxxxxxxxxx
```

Et ajoute `.env` à ton `.gitignore` :

```bash
echo ".env" >> .gitignore
```

```bash
docker compose up --build
```

---

## Générer un portrait de Zeus

Une fois le serveur démarré sur `http://localhost:8000` :

```bash
curl -X POST http://localhost:8000/generate \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A majestic Maine Coon cat in a general uniform, dramatic lighting, digital art",
    "steps": 4,
    "width": 512,
    "height": 512
  }' | python3 -c "
import sys, json, base64
data = json.load(sys.stdin)
with open('zeus_portrait.png', 'wb') as f:
    f.write(base64.b64decode(data['image_base64']))
print('Portrait sauvegardé : zeus_portrait.png')
"
```

Ou en Python, depuis un agent ou n'importe quel script :

```python
import requests
import base64

response = requests.post(
    "http://localhost:8000/generate",
    json={
        "prompt": "A Maine Coon cat wearing a general uniform, dramatic lighting, digital art",
        "steps": 4,
        "width": 512,
        "height": 512,
    }
)

with open("zeus_portrait.png", "wb") as f:
    f.write(base64.b64decode(response.json()["image_base64"]))
```

---

## Pourquoi le CPU offloading est nécessaire

Même sur 16 Go, le modèle dépasse la VRAM disponible pendant l'inférence. Les poids du transformer chargent bien en ~8 Go, mais le **VAE decoder** — qui convertit l'espace latent en image finale — a besoin d'un pic mémoire supplémentaire qui fait déborder la carte.

`enable_model_cpu_offload()` déplace automatiquement chaque composant sur le CPU quand il n'est pas utilisé à une étape donnée : text encoder, transformer, VAE se partagent la VRAM au lieu de tout occuper simultanément. Le pic reste sous les 12 Go.

Contrepartie : chaque image prend **20 à 40 secondes** au lieu de 5 à 10 s. C'est le prix du 16 Go. Zeus accepte. Zeus a la patience des prédateurs.

---

## Ce que j'avais mal compris

**"vLLM gère tout, y compris la génération d'images"** — Non. vLLM est conçu pour les LLM et les modèles vision-langage. Les modèles de diffusion comme FLUX sont un pipeline fondamentalement différent. vLLM-Omni travaille dessus, mais ce n'est pas encore prêt en mars 2026.

**"bfloat16 c'est trop lourd"** — FLUX.2-klein-4B fait ~8 Go en bfloat16 (4B paramètres × 2 octets). Sur 16 Go de VRAM, il reste de la marge pour les activations et le pipeline de diffusion. Pas besoin de quantization.

**"Docker c'est trop lourd pour de l'inférence locale"** — Docker avec NVIDIA Container Toolkit ne rajoute quasiment aucune latence GPU. Le vrai coût, c'est le build initial (~8 Go d'image) — une fois fait, chaque lancement démarre en quelques secondes.

**"Il faut une bête de course"** — FLUX.2-klein-4B en bfloat16 avec CPU offloading tourne sur une RTX 4060 Ti 16 Go en **20 à 40 secondes par image**. Pas fulgurant, mais c'est local, gratuit, et suffisant pour équiper une armée féline.

---

## En une phrase

FLUX.2-klein-4B avec HuggingFace Diffusers en bfloat16 et Docker produit une API locale de génération d'images qui tient sur 16 Go de VRAM sans quantization, s'isole proprement du système hôte, et génère des portraits d'armée féline tout à fait convenables pour une conquête galactique sérieuse.

---

*Zeus a désormais ses visuels. Le Collectif Félin Mondial aussi. L'invasion galactique peut commencer.*
