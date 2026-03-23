# Architecture DeerFlow

## Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT (Navigateur)                             │
│                         http://localhost:2026                                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              NGINX (Reverse Proxy)                          │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┬──────────────┐ │
│  │ /          │ /api/*      │ /docs       │ /health     │ /api/langgraph│ │
│  │ (Frontend) │ (Gateway)   │ (Gateway)   │ (Gateway)   │ (LangGraph)  │ │
│  └─────────────┴─────────────┴─────────────┴─────────────┴──────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
        ┌───────────────────┐ ┌───────────────┐ ┌─────────────────┐
        │    FRONTEND       │ │   GATEWAY     │ │   LANGGRAPH     │
        │   (Next.js)       │ │   (FastAPI)   │ │   (LangGraph)   │
        │   Port 3000       │ │   Port 8001   │ │   Port 2024     │
        └───────────────────┘ └───────────────┘ └─────────────────┘
                                          │                   │
                                          │                   │
                                          ▼                   │
                              ┌───────────────────────┐        │
                              │   CHANNELS          │        │
                              │ ┌─────────────────┐ │        │
                              │ │ • Telegram Bot  │ │        │
                              │ │ • Slack Bot     │ │        │
                              │ │ • Feishu Bot    │ │        │
                              │ └─────────────────┘ │        │
                              └───────────────────────┘        │
                                                             │
                                     ┌────────────────────────┘
                                     ▼
                        ┌─────────────────────────┐
                        │   EXTERNAL SERVICES     │
                        │ ┌───────────────────┐  │
                        │ │ OpenRouter API    │  │
                        │ │ Trinity Large     │  │
                        │ └───────────────────┘  │
                        │ ┌───────────────────┐  │
                        │ │ Telegram API      │  │
                        │ │ (@LobsterBot)    │  │
                        │ └───────────────────┘  │
                        │ ┌───────────────────┐  │
                        │ │ Jina AI Reader   │  │
                        │ └───────────────────┘  │
                        └─────────────────────────┘
```

## Containers Docker

| Container | Image | Port | Description |
|-----------|-------|------|-------------|
| `deer-flow-nginx` | nginx:alpine | 2026 | Reverse proxy, routing des requêtes |
| `deer-flow-frontend` | docker-frontend | 3000 | Interface Next.js |
| `deer-flow-gateway` | docker-gateway | 8001 | API Gateway FastAPI |
| `deer-flow-langgraph` | docker-langgraph | 2024 | Serveur LangGraph |

## Flux des requêtes

### 1. Requête Web (Interface UI)

```
Utilisateur → Nginx:2026 → Frontend:3000 (UI React)
     ↓
  Chat envoyé
     ↓
  Frontend → Nginx → Gateway:8001 → LangGraph:2024
                                    ↓
                              OpenRouter API
                                    ↓
                              Réponse → Utilisateur
```

### 2. Requête Telegram

```
Telegram User → Telegram API → DeerFlow Gateway:8001
                                    ↓
                              LangGraph:2024
                                    ↓
                              OpenRouter API
                                    ↓
                              Telegram Bot Response
```

### 3. Requête API Directe

```
Client API → Nginx:2026 → /api/* → Gateway:8001
                              → /api/langgraph/* → LangGraph:2024
```

## Configuration

### Fichiers principaux

| Fichier | Description |
|---------|-------------|
| `config.yaml` | Configuration des modèles, tools, channels |
| `docker/.env` | Variables d'environnement Docker |
| `docker/docker-compose.yaml` | Définition des services |
| `docker/nginx/nginx.conf` | Configuration du reverse proxy |

### Variables d'environnement critiques

```bash
# API Keys
OPENROUTER_API_KEY=sk-or-v1-xxx    # Pour les modèles LLM
TELEGRAM_BOT_TOKEN=xxx             # Bot Telegram

# Chemins
DEER_FLOW_CONFIG_PATH=/path/to/config.yaml
DEER_FLOW_HOME=/path/to/.deer-flow

# Ports
PORT=2026                          # Port public
```

## Modèle OpenRouter Configuré

```yaml
models:
  - name: arcee-ai/trinity-large-preview:free
    display_name: Trinity Large (Free)
    use: langchain_openai:ChatOpenAI
    model: arcee-ai/trinity-large-preview:free
    api_key: $OPENROUTER_API_KEY
    base_url: https://openrouter.ai/api/v1
    max_tokens: 8192
    supports_thinking: true
```

## Channels (Messagerie)

### Telegram
```yaml
channels:
  telegram:
    enabled: true
    bot_token: $TELEGRAM_BOT_TOKEN
    allowed_users: []  # Vide = tous les utilisateurs
```

## Endpoints API

| Endpoint | Méthode | Description |
|----------|---------|-------------|
| `/api/models` | GET | Liste des modèles disponibles |
| `/api/models/{name}` | GET | Détails d'un modèle |
| `/api/threads` | POST | Créer un thread |
| `/api/channels` | GET | Statut des channels |
| `/api/agents` | GET | Liste des agents |
| `/health` | GET | Health check |
| `/docs` | GET | Swagger UI |

## Commandes Utiles

```bash
# Démarrer DeerFlow
cd /home/ubuntu/Projets_All/deer-flow/docker
docker compose up -d

# Voir les logs
docker logs -f deer-flow-gateway
docker logs -f deer-flow-langgraph

# Redémarrer un service
docker restart deer-flow-gateway

# Arrêter DeerFlow
docker compose down
```

## Troubleshooting

### Gateway ne démarre pas (clé API manquante)
```
RuntimeError: Environment variable OPENROUTER_API_KEY not found
```
→ Vérifier que la clé est dans `docker/.env`

### Telegram Conflict
```
telegram.error.Conflict: terminated by other getUpdates request
```
→ Attendre 5 minutes ou redémarrer le bot

### Modèle non valide
```
openai.BadRequestError: openai/trinity-large-preview is not a valid model ID
```
→ Utiliser le format OpenRouter: `arcee-ai/trinity-large-preview:free`
