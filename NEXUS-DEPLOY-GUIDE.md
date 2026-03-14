# Nexus v2

Dashboard self-hosted pour media stack + Docker. Server-side proxy — un seul port exposé.

## Comment ça marche

```
Internet → NPM (nexus.domain.fr) → Nexus :3000
                                        ├─ /s/sonarr/*   → proxy → http://10.0.0.2:8989/s/sonarr/*
                                        ├─ /s/radarr/*   → proxy → http://10.0.0.2:7878/s/radarr/*
                                        ├─ /s/jellyfin/* → proxy → http://10.0.0.2:8096/s/jellyfin/*
                                        └─ /api/*        → Nexus backend (auth, docker, config)
```

Le navigateur ne contacte jamais les services directement.
C'est le serveur Nexus qui va chercher les pages et te les renvoie.

## Installation

### 1. Cloner et configurer

```bash
git clone <repo> && cd nexus
cp .env.example .env
nano .env
```

Remplir :
```env
JWT_SECRET=<openssl rand -hex 32>
JELLYFIN_URL=http://10.0.0.2:8096
JELLYFIN_API_KEY=<depuis Jellyfin Dashboard → API Keys>
```

### 2. Configurer les URL Base des services

**IMPORTANT** — Chaque service doit avoir son URL Base configuré :

| Service      | Où configurer                           | Valeur          |
|-------------|----------------------------------------|-----------------|
| Sonarr      | Settings → General → URL Base          | `/s/sonarr`     |
| Radarr      | Settings → General → URL Base          | `/s/radarr`     |
| Prowlarr    | Settings → General → URL Base          | `/s/prowlarr`   |
| Jellyseerr  | Settings → General → Base URL          | `/s/jellyseerr` |
| Jellyfin    | Dashboard → Networking → Base URL      | `/s/jellyfin`   |
| Transmission| Variable env `TRANSMISSION_WEB_HOME`   | Voir docs       |

Les services redémarrent après le changement. C'est une config one-time.

### 3. Lancer

```bash
docker compose up -d
```

### 4. Configurer dans Nexus

1. Ouvrir `http://ton-serveur:3000`
2. Se connecter avec un compte Jellyfin admin
3. Paramètres → Services → Renseigner les URLs internes + API keys
4. Paramètres → Hôtes Docker → Ajouter tes serveurs
5. Paramètres → Utilisateurs → Configurer les permissions

### 5. Exposer via NPM (accès distant)

Créer un Proxy Host dans NPM :
- Domain: `nexus.ton-domaine.fr`
- Forward: `IP_NEXUS:3000`
- Websockets: ✅
- SSL: Let's Encrypt

## API Keys des services

| Service | Où trouver |
|---------|-----------|
| Sonarr/Radarr/Prowlarr | Settings → General → API Key |
| Jellyseerr | Settings → General → API Key |
| Jellyfin | Dashboard → API Keys → Créer |
| Transmission | User/Password dans la config du service |

## Sécurité

- Auth Jellyfin SSO avec JWT httpOnly cookies
- Permissions par utilisateur par service
- Rate limiting (300 req/15min, 15/15min sur login)
- Proxy server-side (services jamais exposés)
- Config persistante en JSON avec écriture atomique

## Mise à jour

```bash
git pull && docker compose build --no-cache && docker compose up -d
```
