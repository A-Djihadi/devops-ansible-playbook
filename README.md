# devops-ansible-playbook

Template **Ansible playbook** avec roles, group_vars et vault — structure production-ready prête à cloner et adapter.

---

## Démarrage en 5 étapes / Quick start

```bash
# 1. Cloner le template
git clone https://github.com/A-Djihadi/devops-ansible-playbook.git
cd devops-ansible-playbook

# 2. Adapter l'inventaire avec tes serveurs réels
vim inventory/hosts.yml

# 3. Créer et chiffrer le vault des secrets
cp vault/secrets.yml.example vault/secrets.yml
vim vault/secrets.yml          # remplir les vraies valeurs
ansible-vault encrypt vault/secrets.yml

# 4. Tester la connexion SSH
ansible all -i inventory/hosts.yml -m ping

# 5. Lancer le playbook
ansible-playbook -i inventory/hosts.yml playbook.yml --ask-vault-pass
```

---

## Structure

```
devops-ansible-playbook/
├── inventory/
│   ├── hosts.yml              # Inventaire (remplacer les IPs fictives)
│   └── group_vars/
│       ├── all.yml            # Variables communes
│       └── webservers.yml     # Variables du groupe webservers
├── playbook.yml               # Point d'entrée principal
├── roles/
│   ├── common/                # Paquets système, timezone, user applicatif
│   ├── docker/                # Installation Docker Engine + Compose plugin
│   └── app-deploy/            # Déploiement via docker-compose (template Jinja2)
└── vault/
    └── secrets.yml.example    # Modèle — jamais de vrais secrets ici
```

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

Dans les variables, les secrets sont référencés avec la convention `vault_` :

```yaml
# group_vars/webservers.yml
app_registry_token: "{{ vault_registry_token }}"
```

---

## Idempotence

Tous les playbooks sont idempotents : les relancer plusieurs fois produit le même résultat sans effets de bord. Pour vérifier :

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
