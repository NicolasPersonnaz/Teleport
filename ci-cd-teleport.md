🚀 Teleport-CI : Le Guide Ultime du Déploiement "Zero Trust"

Cette documentation détaille comment automatiser un déploiement sécurisé depuis GitLab CI vers un VPS distant via Teleport v18, en utilisant l'authentification OIDC (sans clés SSH statiques) et un utilisateur Livreur (Least Privilege).

1. L'Image Docker CI (La version "Light")

Nous utilisons une image Debian Slim embarquant les binaires Teleport officiels pour garantir un environnement de déploiement léger et maîtrisé.

Préparation des binaires (AMD64)

# Téléchargement de Teleport
wget [https://cdn.teleport.dev/teleport-v18.3.2-linux-amd64-bin.tar.gz](https://cdn.teleport.dev/teleport-v18.3.2-linux-amd64-bin.tar.gz)
tar -xzf teleport-v18.3.2-linux-amd64-bin.tar.gz

# Extraction des outils nécessaires
mv teleport/teleport ./teleport_bin
mv teleport/tsh ./tsh
mv teleport/tbot ./tbot


Dockerfile

FROM debian:bullseye-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    rsync ca-certificates bash && rm -rf /var/lib/apt/lists/*

COPY teleport_bin /usr/local/bin/teleport
COPY tsh /usr/local/bin/tsh
COPY tbot /usr/local/bin/tbot

RUN chmod +x /usr/local/bin/teleport /usr/local/bin/tsh /usr/local/bin/tbot


Build & Push : docker build --platform linux/amd64 -t registry.votre-domaine.fr/infra/teleport-ci:light .

2. Configuration Teleport (RBAC)

Il est nécessaire de configurer deux rôles : un pour le Bot (qui s'authentifie via OIDC) et un pour le Livreur (qui exécute les commandes sur le serveur).

Rôle du Livreur (deployer-role.yaml)

Ce rôle autorise uniquement rsync via sudo et gère la création automatique de l'utilisateur système.

kind: role
version: v5
metadata:
  name: deployer-role
spec:
  allow:
    logins: [deployer]
    node_labels:
      'access': 'production'
    host_sudoers:
    - 'ALL=(ALL) NOPASSWD: /usr/bin/rsync'
  options:
    create_host_user: true
    create_host_user_mode: keep
    ssh_file_copy: true


Rôle du Bot (bot-gitlab-bot.yaml)

Il permet au bot d'utiliser l'identité du rôle livreur.

kind: role
version: v5
metadata:
  name: bot-gitlab-bot
spec:
  allow:
    impersonate:
      roles: [deployer-role]


3. Configuration GitLab CI (.gitlab-ci.yml)

Le pipeline utilise le jeton OIDC de GitLab pour s'authentifier dynamiquement auprès de Teleport.

stages:
  - check
  - security
  - deploy

variables:
  TOOLS_IMAGE: "registry.votre-domaine.fr/devops/pythontools:latest"
  DEPLOY_IMAGE: "registry.votre-domaine.fr/infra/teleport-ci:light"

# --- QUALITÉ DU CODE ---
ruff-lint:
  stage: check
  image: $TOOLS_IMAGE
  script:
    - pip install ruff --root-user-action ignore
    - ruff check .
  rules:
    - when: always

# --- SÉCURITÉ ---
gitleaks:
  stage: security
  image: $TOOLS_IMAGE
  before_script:
    - curl -sSL [https://github.com/gitleaks/gitleaks/releases/download/v8.18.2/gitleaks_8.18.2_linux_x64.tar.gz](https://github.com/gitleaks/gitleaks/releases/download/v8.18.2/gitleaks_8.18.2_linux_x64.tar.gz) | tar -xz -C /usr/local/bin gitleaks
  script:
    - gitleaks detect --source . -c .gitleaks.toml --verbose --redact
  rules:
    - when: always 

sast-semgrep:
  stage: security
  image: returntocorp/semgrep
  script:
    - semgrep scan --config auto --error .
  rules:
    - when: always 

# --- DÉPLOIEMENT ---
deploy-to-srv:
  stage: deploy
  image: $DEPLOY_IMAGE
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: teleport.votre-domaine.fr
  script:
    - |
      set -e
      if [ -z "$TELEPORT_PROXY" ]; then echo "ERREUR: TELEPORT_PROXY vide"; exit 1; fi
      mkdir -p /tmp/tbot

      # 1. Génération de la configuration tbot
      cat <<EOF > tbot-config.yaml
      version: v2
      instance_addr: ${TELEPORT_PROXY}
      onboarding:
        join_method: gitlab
        token: gitlab-oidc-token
      storage:
        type: memory
      outputs:
        - type: identity
          destination:
            type: directory
            path: /tmp/tbot
      EOF

      # 2. Authentification
      export TBOT_GITLAB_JWT="${GITLAB_OIDC_TOKEN}"
      tbot start -c tbot-config.yaml --oneshot
      export TELEPORT_AUTH=local

      # 3. Préparation du dossier cible sur le serveur
      echo "🔧 Configuration de l'environnement sur ${TARGET_SERVER}..."
      tsh ssh -i /tmp/tbot/identity --proxy=${TELEPORT_PROXY} ${TARGET_SERVER} "sudo mkdir -p /srv/app && sudo chown deployer:deployer /srv/app"

      # 4. Transfert des fichiers via SCP sécurisé
      echo "📦 Transfert des artefacts..."
      tsh scp -i /tmp/tbot/identity --proxy=${TELEPORT_PROXY} -r ./* ${TARGET_SERVER}:/srv/app/
      
      echo "✅ Livraison terminée !"


4. Variables CI/CD requises (GitLab)

Variable

Valeur exemple

Description

TELEPORT_PROXY

teleport.votre-domaine.fr:443

Adresse publique de votre cluster Teleport

TARGET_SERVER

deployer@node-prod-01

Format user@node-name configuré dans Teleport

🛡️ Pourquoi cette architecture ?

Pas de clés statiques : Si votre instance GitLab est compromise, il n'y a aucune clé SSH à voler (l'accès dépend d'un jeton OIDC éphémère).

Utilisateur éphémère : L'utilisateur Linux deployer est géré par Teleport, limitant les risques de comptes "fantômes" sur les serveurs.

Auditabilité : Chaque action de déploiement est enregistrée dans l'Audit Log de Teleport, associée à l'ID du job GitLab pour une traçabilité totale.
