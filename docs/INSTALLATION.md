# Installation DeerFlow - Guide Complet

## Prérequis

- Docker & Docker Compose
- Clé API OpenRouter (optionnel, pour les modèles LLM)
- Token Bot Telegram (optionnel, pour le channel Telegram)

## Installation Rapide

### 1. Cloner le repository

```bash
cd /home/ubuntu/Projets_All
git clone https://github.com/bytedance/deer-flow.git
cd deer-flow
```

### 2. Configurer les variables d'environnement

```bash
# Créer le fichier .env principal
cat > .env << 'EOF'
# TAVILY_API Key (optionnel - pour recherche web)
TAVILY_API_KEY=your-tavily-api-key

# Jina API Key (optionnel - pour lecture de pages)
JINA_API_KEY=your-jina-api-key

# OpenRouter API Key (pour les modèles LLM)
OPENROUTER_API_KEY=sk-or-v1-your-key

# Telegram Bot Token (optionnel)
TELEGRAM_BOT_TOKEN=your-telegram-bot-token
EOF
```

### 3. Configurer le modèle LLM

```bash
# Éditer config.yaml et ajouter/modifier la section models:
cat >> config.yaml << 'EOF'

models:
  - name: arcee-ai/trinity-large-preview:free
    display_name: Trinity Large (Free)
    use: langchain_openai:ChatOpenAI
    model: arcee-ai/trinity-large-preview:free
    api_key: $OPENROUTER_API_KEY
    base_url: https://openrouter.ai/api/v1
    max_tokens: 8192
    temperature: 0.7
    supports_thinking: true
    supports_vision: true
EOF
```

### 4. Configurer Docker

```bash
cd docker

# Créer le fichier .env Docker
cat > .env << 'EOF'
PORT=2026
NGINX_CONF=nginx.conf
BETTER_AUTH_SECRET=your-random-secret-here

DEER_FLOW_HOME=/path/to/deer-flow/backend/.deer-flow
DEER_FLOW_CONFIG_PATH=/path/to/deer-flow/config.yaml
DEER_FLOW_EXTENSIONS_CONFIG_PATH=/path/to/deer-flow/extensions_config.json
DEER_FLOW_DOCKER_SOCKET=/var/run/docker.sock
DEER_FLOW_REPO_ROOT=/path/to/deer-flow

# API Keys
OPENROUTER_API_KEY=sk-or-v1-your-key
TELEGRAM_BOT_TOKEN=your-telegram-bot-token
EOF
```

### 5. Activer Telegram (optionnel)

```bash
# Dans config.yaml, décommenter et configurer:
channels:
  telegram:
    enabled: true
    bot_token: $TELEGRAM_BOT_TOKEN
    allowed_users: []  # Vide = tous les utilisateurs
```

### 6. Démarrer DeerFlow

```bash
# Construire et démarrer
docker compose up -d --build

# Ou sans build (si déjà construit)
docker compose up -d

# Vérifier le statut
docker ps | grep deer-flow
```

### 7. Vérifier l'installation

```bash
# Health check
curl http://localhost:2026/health

# Voir les logs
docker logs -f deer-flow-gateway
docker logs -f deer-flow-langgraph
```

## Accès

| Service | URL |
|---------|-----|
| Interface Web | http://localhost:2026 |
| API Docs | http://localhost:2026/docs |
| Modèles | http://localhost:2026/api/models |

## Commandes de Gestion

```bash
# Démarrer
docker compose start

# Arrêter
docker compose stop

# Redémarrer
docker compose restart

# Recommencer depuis zéro
docker compose down -v
docker compose up -d --build

# Voir tous les logs
docker compose logs -f

# Voir logs d'un service
docker logs -f deer-flow-gateway
docker logs -f deer-flow-langgraph
docker logs -f deer-flow-frontend
```

## Dépannage

### Le gateway ne démarre pas

```bash
# Vérifier les logs
docker logs deer-flow-gateway

# Erreur "Environment variable not found"
→ Vérifier que les variables sont dans docker/.env

# Redémarrer
docker restart deer-flow-gateway
```

### L'interface affiche une erreur

```bash
# Vérifier le frontend
docker logs deer-flow-frontend

# Vérifier la connexion au gateway
curl http://localhost:2026/api/models
```

### Le modèle ne fonctionne pas

```bash
# Vérifier le modèle configuré
curl http://localhost:2026/api/models

# Vérifier les logs langgraph
docker logs deer-flow-langgraph | grep -i error
```

### Telegram en conflit

```bash
# Attendre 5 minutes et redémarrer
docker restart deer-flow-gateway
```

## Structure des Fichiers

```
deer-flow/
├── config.yaml              # Configuration principale (modèles, tools)
├── extensions_config.json   # Configuration extensions
├── docker/
│   ├── .env                # Variables d'environnement Docker
│   ├── docker-compose.yaml  # Définition des services
│   └── nginx/
│       └── nginx.conf      # Configuration reverse proxy
├── frontend/                # Interface Next.js
│   └── .env
├── backend/                # Code Python (gateway, agents)
│   └── .deer-flow/        # Données runtime
└── docs/
    ├── ARCHITECTURE.md     # Architecture système
    └── INSTALLATION.md     # Ce fichier
```

## Mise à jour

```bash
cd /home/ubuntu/Projets_All/deer-flow
git pull
cd docker
docker compose down
docker compose up -d --build
```

## Désinstallation

```bash
cd /home/ubuntu/Projets_All/deer-flow/docker
docker compose down -v
cd ..
rm -rf backend/.deer-flow
```
