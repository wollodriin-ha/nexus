# Nexus

Dashboard self-hosted pour gérer ton media stack et ton infrastructure Docker.
Auth centralisée via Jellyfin, gestion multi-hôte Docker, proxy iframe sécurisé pour chaque service.

---

## Architecture

```
┌─────────────┐     HTTPS      ┌──────────────┐
│   Browser    │ ◄────────────► │  NPM / Nginx │
└─────────────┘                 └──────┬───────┘
                                       │ :3000
                                ┌──────▼───────┐
                                │  Nexus Node  │
                                │  (Express)   │
                                └──┬───┬───┬───┘
                    ┌──────────────┤   │   ├────────────────┐
                    ▼              ▼   │   ▼                ▼
              ┌──────────┐  ┌─────────┐│ ┌──────────┐ ┌──────────┐
              │ Jellyfin │  │ Sonarr  ││ │ Radarr   │ │ Docker   │
              │ (SSO)    │  │         ││ │          │ │ API TCP  │
              └──────────┘  └─────────┘│ └──────────┘ └──────────┘
                                       │
                              ┌────────▼────────┐
                              │  Prowlarr       │
                              │  Jellyseerr     │
                              │  Transmission   │
                              │  + custom...    │
                              └─────────────────┘
```

---

## Prérequis

- Node.js >= 20
- Docker & Docker Compose
- Un serveur Jellyfin fonctionnel
- Une clé API Jellyfin (Dashboard > API Keys)

---

## Installation rapide (Docker)

### 1. Cloner le repo

```bash
git clone https://github.com/ton-user/nexus.git
cd nexus
```

### 2. Configurer l'environnement

```bash
cp .env.example .env
```

Éditer `.env` :

```env
# IMPORTANT: Générer un vrai secret
JWT_SECRET=$(openssl rand -hex 32)

# L'URL de ton Jellyfin (accessible depuis le serveur Nexus)
JELLYFIN_URL=http://10.0.0.2:8096

# Clé API Jellyfin: Dashboard > API Keys > +
JELLYFIN_API_KEY=ta_cle_api_ici
```

### 3. Lancer

```bash
docker compose up -d
```

Nexus est accessible sur `http://ton-serveur:3000`.

---

## Installation manuelle (sans Docker)

```bash
# Installer les dépendances et build le frontend
npm run setup

# Configurer
cp .env.example .env
nano .env  # remplir JWT_SECRET, JELLYFIN_URL, JELLYFIN_API_KEY

# Lancer en production
npm start
```

Pour le développement :

```bash
npm run dev
# → Backend sur :3000, Frontend sur :5173 (avec proxy auto)
```

---

## Configuration post-installation

### 1. Première connexion

Ouvre `http://ton-serveur:3000` et connecte-toi avec ton compte Jellyfin admin.

### 2. Configurer les services

Va dans **Paramètres > Services** et renseigne l'URL + clé API de chaque service :

| Service      | URL par défaut            | Clé API                              |
|-------------|---------------------------|--------------------------------------|
| Jellyfin    | http://10.0.0.2:8096      | Configurée dans .env (SSO)           |
| Sonarr      | http://10.0.0.2:8989      | Settings > General > API Key         |
| Radarr      | http://10.0.0.2:7878      | Settings > General > API Key         |
| Prowlarr    | http://10.0.0.2:9696      | Settings > General > API Key         |
| Jellyseerr  | http://10.0.0.2:5055      | Settings > General > API Key         |
| Transmission| http://10.0.0.2:9091      | (user:password dans l'URL si auth)   |

### 3. Configurer les hôtes Docker

Va dans **Paramètres > Hôtes Docker** et ajoute tes serveurs.

**Pour chaque hôte, tu dois exposer l'API Docker en TCP :**

Sur le serveur cible, éditer `/etc/docker/daemon.json` :

```json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"]
}
```

Puis `sudo systemctl restart docker`.

**⚠ SÉCURITÉ** : L'API Docker TCP sans TLS est dangereuse en réseau ouvert.
Options recommandées :
- Utiliser TLS (méthode "TCP TLS" dans Nexus)
- Restreindre l'accès via firewall (iptables/pfSense)
- Utiliser un tunnel WireGuard entre Nexus et les hôtes

Pour ton Pi via WireGuard, utilise l'IP WireGuard du Pi comme adresse.

### 4. Configurer les permissions

Va dans **Paramètres > Utilisateurs & Accès**.
Les utilisateurs Jellyfin sont automatiquement synchronisés.
Active/désactive l'accès par service et par utilisateur.

---

## Mise derrière Nginx Proxy Manager (NPM)

Dans NPM, crée un nouveau Proxy Host :

- **Domain**: `nexus.ton-domaine.fr`
- **Forward Hostname**: `IP_du_serveur_nexus` ou `nexus` (si même réseau Docker)
- **Forward Port**: `3000`
- **Websockets Support**: ✅
- **Block Common Exploits**: ✅
- **SSL**: Let's Encrypt, Force SSL

Si Nexus et NPM sont sur le même serveur Docker, ajoute Nexus au réseau NPM :

```yaml
# docker-compose.yml
services:
  nexus:
    # ...
    networks:
      - nexus-net
      - npm_default  # Le réseau de ton NPM

networks:
  nexus-net:
  npm_default:
    external: true
```

---

## Mise derrière Teleport

Si tu préfères utiliser Teleport (bst.jules-ropers.fr) :

```yaml
# /etc/teleport.yaml (ajout app_service)
app_service:
  enabled: yes
  apps:
    - name: nexus
      uri: http://localhost:3000
      public_addr: nexus.bst.jules-ropers.fr
```

---

## Structure du projet

```
nexus/
├── server/
│   ├── index.js              # Point d'entrée Express
│   ├── config.js             # Config JSON persistante
│   ├── logger.js             # Winston logger
│   ├── middleware/
│   │   └── auth.js           # JWT + permissions
│   ├── routes/
│   │   ├── auth.js           # Login/logout Jellyfin SSO
│   │   ├── docker.js         # Gestion Docker multi-hôte
│   │   ├── proxy.js          # Reverse proxy iframe
│   │   └── settings.js       # CRUD services/hosts/perms
│   └── services/
│       ├── jellyfin.js       # Client API Jellyfin
│       └── docker.js         # Client Dockerode multi-hôte
├── client/
│   ├── src/
│   │   ├── App.jsx           # Frontend React complet
│   │   ├── main.jsx          # Entry point
│   │   └── lib/
│   │       └── api.js        # Client API fetch
│   ├── index.html
│   ├── vite.config.js
│   └── package.json
├── data/                      # Config persistante (volume Docker)
│   └── config.json           # Généré automatiquement
├── Dockerfile                # Multi-stage build
├── docker-compose.yml
├── package.json
├── .env.example
└── .gitignore
```

---

## Sécurité

- **JWT httpOnly cookies** — pas de token en localStorage
- **Rate limiting** — 200 req/15min global, 15/15min sur /login
- **Helmet** — headers de sécurité (CSP, HSTS, etc.)
- **API keys côté serveur** — jamais exposées au client (sauf admin)
- **Permissions par service** — matrice user × service stockée côté serveur
- **Proxy authentifié** — chaque iframe passe par le backend, pas d'accès direct
- **Atomic config writes** — pas de corruption de config.json

---

## FAQ

**Q: Les iframes des services ne chargent pas ?**
Le backend supprime les headers `X-Frame-Options` des réponses. Si un service persiste,
vérifie sa config interne (Sonarr/Radarr n'ont normalement pas de restriction iframe).

**Q: Comment ajouter un nouveau service custom ?**
Paramètres > Services > "Ajouter un service". Choisis nom, port, icône, couleur.
Le service apparaît immédiatement dans la sidebar.

**Q: Comment faire les mises à jour Docker ?**
Clique sur l'icône ↑ (Pull & Recreate) dans la vue Docker.
Ça fait `docker pull image:latest` puis recréé le conteneur avec la même config.
Si le conteneur est dans un docker-compose, utilise le bouton compose.

**Q: Comment mettre à jour Nexus lui-même ?**
```bash
git pull
docker compose build
docker compose up -d
```

---

## Licence

MIT
