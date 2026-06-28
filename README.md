# ansible-role-element-web

Ansible role to deploy [Element Web](https://element.io/) — the reference Matrix client — as a Docker Compose service on Debian 12.

## Architecture

```
Internet
   │
Reverse Proxy (Caddy — géré dans devops_staging_prod_infra)
   │
   ├── element.staging.lavallee.tech  →  Element Web (ce rôle, port 8080)
   ├── matrix-users.staging.lavallee.tech  →  Synapse Users (ansible-role-matrix-stack)
   └── matrix-bridges.staging.lavallee.tech  →  Synapse Bridges (ansible-role-matrix-stack)
```

### Séparation des responsabilités

| Rôle | Responsabilité |
|------|---------------|
| **ansible-role-element-web** (ce dépôt) | Client web Element uniquement |
| [ansible-role-matrix-stack](https://github.com/sepp67/ansible-role-matrix-stack) | PostgreSQL + Synapse Users + Synapse Bridges |
| [devops_staging_prod_infra](https://github.com/sepp67/devops_staging_prod_infra) | Inventaires, Vault, proxy Caddy, orchestration staging/production |

Element est volontairement découplé du backend Matrix pour permettre :

- des cycles de déploiement indépendants ;
- un changement de homeserver sans retoucher le client ;
- une réutilisation du rôle sur d'autres infrastructures.

## Ce que fait ce rôle

- Installe Docker Engine + Docker Compose v2 (désactivable)
- Crée `/opt/element-web/` avec les permissions correctes
- Génère `docker-compose.yml` et `config.json` depuis des templates Jinja2
- Démarre le conteneur `vectorim/element-web` via `docker compose`
- Recharge la configuration à chaque changement (handler idempotent)

## Ce que ce rôle ne fait pas

- Ne touche pas à Synapse ni à PostgreSQL
- Ne configure pas de reverse proxy (Caddy/nginx)
- Ne gère pas TLS
- Ne crée pas d'utilisateurs Matrix

## Prérequis

- Debian 12 (Bookworm)
- Ansible >= 2.14
- Collection `community.docker >= 3.0.0`

```bash
ansible-galaxy collection install -r requirements.yml
```

## Installation du rôle

### Via requirements.yml (recommandé)

```yaml
roles:
  - src: https://github.com/sepp67/ansible-role-element-web.git
    scm: git
    version: main
    name: element_web
```

```bash
ansible-galaxy role install -r requirements.yml
```

### Clone direct

```bash
git clone https://github.com/sepp67/ansible-role-element-web.git roles/element_web
```

## Variables

### Docker

| Variable | Défaut | Description |
|----------|--------|-------------|
| `element_install_docker` | `true` | Installer Docker automatiquement |
| `docker_bind_address` | `0.0.0.0` | Adresse d'écoute des ports Docker |
| `deploy_user` | `devops` | Utilisateur système pour Docker Compose |
| `environment_name` | `staging` | Nom de l'environnement (commentaires) |

### Image et conteneur

| Variable | Défaut | Description |
|----------|--------|-------------|
| `element_image` | `vectorim/element-web` | Image Docker officielle |
| `element_version` | `latest` | Tag de l'image |
| `element_container_name` | `element-web` | Nom du conteneur |
| `element_http_port` | `8080` | Port HTTP exposé sur l'hôte |
| `element_base_dir` | `/opt/element-web` | Répertoire de déploiement |

### Configuration Element

| Variable | Défaut | Description |
|----------|--------|-------------|
| `element_default_homeserver_url` | `https://matrix-users.example.com` | URL publique du homeserver Matrix |
| `element_default_homeserver_name` | `matrix-users.example.com` | `server_name` du homeserver |
| `element_disable_custom_urls` | `false` | Verrouiller le homeserver dans l'UI |
| `element_disable_guests` | `true` | Interdire les connexions invités |
| `element_brand` | `Element` | Nom affiché dans l'interface |
| `element_default_theme` | `light` | Thème par défaut (`light` ou `dark`) |
| `element_additional_homeservers` | `[]` | Homeservers supplémentaires (liste optionnelle) |

### Homeservers additionnels (optionnel)

```yaml
element_additional_homeservers:
  - base_url: "https://matrix-bridges.example.com"
    server_name: "matrix-bridges.example.com"
```

## Exemple de playbook

```yaml
- hosts: element_web
  become: true
  roles:
    - role: element_web
```

## Exemple d'inventaire staging

```yaml
# inventories/staging/hosts.yml
all:
  children:
    element_web:
      hosts:
        vm-element-staging:
          ansible_host: 192.168.1.80
          ansible_user: devops
```

```yaml
# inventories/staging/group_vars/all/main.yml
environment_name: staging
deploy_user: devops

element_default_homeserver_url: "https://matrix-users.staging.lavallee.tech"
element_default_homeserver_name: "matrix-users.staging.lavallee.tech"
element_brand: "Element — Staging"
```

## Validation du service

### Via Docker Compose

```bash
docker compose -f /opt/element-web/docker-compose.yml ps
```

### En local sur la VM

```bash
curl http://localhost:8080/config.json
```

### Via le reverse proxy

```bash
curl https://element.staging.lavallee.tech/config.json
```

## Intégration dans devops_staging_prod_infra

Après publication de ce rôle, les étapes dans `devops_staging_prod_infra` sont :

**1. Ajouter la dépendance dans `requirements.yml`**

```yaml
roles:
  - src: https://github.com/sepp67/ansible-role-element-web.git
    scm: git
    version: main
    name: element_web
```

**2. Créer le groupe dans l'inventaire staging**

```yaml
# inventories/staging/hosts.yml
element_web:
  hosts:
    vm-element-staging:
      ansible_host: <IP>
```

**3. Ajouter les variables dans `group_vars`**

```yaml
# inventories/staging/group_vars/element_web/main.yml
element_default_homeserver_url: "https://matrix-users.staging.lavallee.tech"
element_default_homeserver_name: "matrix-users.staging.lavallee.tech"
element_brand: "Element — Staging"
```

**4. Ajouter l'entrée dans le Caddyfile**

```
element.staging.lavallee.tech {
    reverse_proxy <IP_VM>:8080
}
```

**5. Créer le playbook**

```yaml
# playbooks/deploy-element-web.yml
- name: Deploy Element Web
  hosts: element_web
  become: true
  roles:
    - role: element_web
```

```bash
ansible-playbook -i inventories/staging/hosts.yml playbooks/deploy-element-web.yml --ask-vault-pass
```

## License

MIT
