# devops-ansible-playbook

Playbook **Ansible** pour provisionner un serveur Ubuntu vierge (Docker + firewall UFW) et déployer une stack applicative multi-services derrière **Traefik**.

> **Objectif :** provisionner un serveur Ubuntu vierge et déployer la stack complète (API Spring Boot + 2 frontends Angular + proxy Traefik) via des rôles Ansible réutilisables.

---

## Démarrage en 6 étapes / Quick start

```bash
# 1. Cloner le repo
git clone https://github.com/A-Djihadi/devops-ansible-playbook.git
cd devops-ansible-playbook

# 2. Installer les collections Ansible requises
ansible-galaxy collection install -r requirements.yml

# 3. Adapter l'inventaire avec tes serveurs réels
vim inventory/hosts.yml            # inventaire de démo (YAML)
# ou
vim inventories/production/hosts.ini
vim inventories/staging/hosts.ini

# 4. Créer et chiffrer le vault des secrets (token GHCR)
cp vault/secrets.yml.example vault/secrets.yml
vim vault/secrets.yml              # renseigner vault_registry_token
ansible-vault encrypt vault/secrets.yml

# 5. Tester la connexion SSH
ansible all -i inventory/hosts.yml -m ping

# 6. Lancer le playbook
ansible-playbook -i inventory/hosts.yml playbook.yml --ask-vault-pass
```

---

## Structure

```text
devops-ansible-playbook/
├── ansible.cfg                 # Configuration Ansible
├── requirements.yml            # Collections Ansible à installer
├── playbook.yml                # Point d'entrée principal (provision + déploiement)
├── group_vars/
│   ├── all.yml                 # Variables communes (GHCR, paquets, UFW)
│   └── webservers.yml          # Variables webservers (env, domain, tags d'images)
├── inventory/
│   └── hosts.yml               # Inventaire de démo (YAML)
├── inventories/
│   ├── production/
│   │   ├── hosts.ini           # Inventaire production
│   │   └── group_vars/webservers.yml
│   └── staging/
│       ├── hosts.ini           # Inventaire staging
│       └── group_vars/webservers.yml
├── roles/
│   ├── common/                 # Paquets système, timezone, user, firewall UFW
│   ├── docker/                 # Installation Docker Engine + Compose plugin
│   └── app-deploy/             # Login GHCR, pull des images, docker compose up
└── vault/
    └── secrets.yml.example     # Modèle — jamais de vrais secrets ici
```

---

## Stack applicative (Traefik multi-services)

Le rôle `app-deploy` déploie 4 services Docker Compose sur le réseau `devops-network` :

| Service   | Image                                            | Rôle                                        |
|-----------|--------------------------------------------------|---------------------------------------------|
| `api`     | `ghcr.io/a-djihadi/devops-spring-app`            | API Spring Boot                             |
| `shell`   | `ghcr.io/a-djihadi/devops-angular-shell`         | Frontend Angular (shell micro-frontend)     |
| `remote`  | `ghcr.io/a-djihadi/devops-angular-remote`        | Frontend Angular (remote micro-frontend)    |
| `proxy`   | `traefik:v3.0`                                   | Reverse-proxy / ingress                     |

Traefik route par nom d'hôte grâce aux labels Docker :

- `api.{{ domain }}` → service `api`
- `{{ domain }}` → service `shell`
- `remote.{{ domain }}` → service `remote`

Le domaine, l'environnement (`app_env`, `spring_profile`) et les tags d'images
(`api_image_tag`, `shell_image_tag`, `remote_image_tag`) sont définis dans
`group_vars/webservers.yml` et surchargés par inventaire.

> **Note TLS :** la configuration Traefik livrée (`roles/app-deploy/templates/traefik.yml.j2`)
> est minimale (HTTP sur `web`). Une fois un vrai domaine et le DNS en place, ajouter un
> bloc `certificatesResolvers` (Let's Encrypt) et basculer les routeurs sur l'entrypoint `websecure`.

---

## Login GHCR

Le rôle `app-deploy` s'authentifie au registry GitHub Container Registry avant de
tirer les images privées :

```yaml
# roles/app-deploy/tasks/main.yml
- name: Login au registry GHCR
  community.docker.docker_login:
    registry_url: "{{ ghcr_registry }}"
    username: "{{ ghcr_owner }}"
    password: "{{ vault_registry_token }}"
```

Le token est lu depuis le vault (`vault/secrets.yml`, clé `vault_registry_token`) :
il doit s'agir d'un token GitHub disposant du scope `read:packages`.

---

## Tags disponibles

```bash
# Uniquement le provisioning (common + docker)
ansible-playbook -i inventory/hosts.yml playbook.yml --tags provision

# Uniquement le déploiement de l'app
ansible-playbook -i inventory/hosts.yml playbook.yml --tags deploy --ask-vault-pass

# Lister toutes les tâches sans les exécuter
ansible-playbook -i inventory/hosts.yml playbook.yml --list-tasks
```

---

## Gestion des secrets avec Ansible Vault

```bash
# Chiffrer un fichier
ansible-vault encrypt vault/secrets.yml

# Déchiffrer temporairement pour édition
ansible-vault edit vault/secrets.yml

# Voir le contenu sans déchiffrer sur disque
ansible-vault view vault/secrets.yml

# Changer le mot de passe du vault
ansible-vault rekey vault/secrets.yml
```

Les secrets sont référencés avec la convention `vault_` (ex. `vault_registry_token`).

---

## Idempotence

Tous les playbooks sont idempotents : les relancer plusieurs fois produit le même
résultat sans effets de bord. Pour vérifier :

```bash
# Première exécution : "changed" pour les tâches modifiées
ansible-playbook -i inventory/hosts.yml playbook.yml

# Deuxième exécution sans changement : toutes les tâches sont "ok"
ansible-playbook -i inventory/hosts.yml playbook.yml
# → PLAY RECAP : ok=N changed=0 unreachable=0 failed=0
```

---

## Mode dry-run

```bash
# Voir ce qui changerait sans rien modifier
ansible-playbook -i inventory/hosts.yml playbook.yml --check --diff
```

---

*Auteur : [Djihadi Ahamdy](https://djihadi.dev) — partie de la [boîte à outils DevOps](https://djihadi.dev)*
