# README de Déploiement - Predil'ection

## 📋 Contexte et Projet

**Predil'ection** est une application web de prédiction électorale développée dans le cadre d'une formation Développeur Data IA à Simplon. L'application permet de :

- Prédire les résultats électoraux d'une commune française
- Consulter les données historiques des élections (2012, 2017)
- Visualiser les données sur une carte interactive
- Authentifier les utilisateurs de manière sécurisée

L'application combine une interface Django pour le frontend, une API FastAPI pour le backend avec des modèles de machine learning (XGBoost, scikit-learn), et une base de données PostgreSQL pour le stockage des données.

---

## 🔧 Prérequis

### Sur le VPS
- **Système d'exploitation** : Linux (Ubuntu 22.04 LTS recommandé)
- **Docker** : version 24.0 ou supérieure
- **Docker Compose** : version 2.20 ou supérieure
- **Git** : pour cloner le repository
- **OpenSSH Server** : pour l'accès distant
- **GitHub CLI (gh)** : pour la configuration des secrets
- **Minimum 2 Go de RAM** (4 Go recommandés)
- **Minimum 20 Go d'espace disque**

### Pour le développement local
- **Python** 3.12
- **PostgreSQL** 15
- **pip** pour la gestion des dépendances

---

## 🏗️ Architecture Retenue

L'application utilise une architecture microservices conteneurisée avec Docker :

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│   Django    │────────▶│   FastAPI   │────────▶│  PostgreSQL │
│   :8000     │         │   :8080     │         │   :5433     │
└─────────────┘         └─────────────┘         └─────────────┘
     Frontend              Backend API            Base de données
                                                   (ML models)
```

### Stack Technique
- **Frontend** : Django 6.0 avec HTML/CSS (framework Bulma)
- **Backend API** : FastAPI 0.135 avec SQLAlchemy ORM
- **Base de données** : PostgreSQL 15
- **Machine Learning** : XGBoost 3.2, scikit-learn 1.8
- **Conteneurisation** : Docker + Docker Compose
- **Registry** : GitHub Container Registry (ghcr.io)
- **Web Server** : Gunicorn (3 workers)
- **Static Files** : Whitenoise

### Flux de Données
1. **Ingest** : Service d'ingestion des données électorales (exécution unique)
2. **FastAPI** : API RESTful avec endpoints de prédiction ML
3. **Django** : Interface utilisateur et authentification
4. **PostgreSQL** : Stockage persistant des données

---

## 🚀 Services Déployés et Ports Utilisés

| Service | Image Docker | Port Hôte | Port Container | Description |
|---------|--------------|-----------|----------------|-------------|
| **PostgreSQL** | postgres:15 | 5433 | 5432 | Base de données |
| **FastAPI** | ghcr.io/nichiku/political-prediction-fastapi:latest | 8080 | 8080 | API Backend avec ML |
| **Django** | ghcr.io/nichiku/political-prediction-django:latest | 8000 | 8000 | Frontend Web |
| **Ingest** | ghcr.io/nichiku/political-prediction-ingest:latest | - | - | Ingestion des données (one-shot) |

**Note** : Le service Ingest ne s'exécute qu'une seule fois pour peupler la base de données initiale.

---

## 📦 Procédure d'Installation sur un VPS Neuf

### 1. Première connexion au VPS

Connectez-vous à votre VPS avec l'utilisateur par défaut (ubuntu) :

```bash
ssh ubuntu@164.132.43.250
```

### 2. Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

### 3. Création de l'utilisateur de déploiement

Créez un utilisateur dédié au déploiement avec les droits sudo :

```bash
sudo useradd -m deployer
sudo passwd deployer
sudo usermod -aG sudo deployer
```

### 4. Mise en place de l'accès SSH par clé

Depuis votre machine locale, générez une paire de clés SSH :

```bash
ssh-keygen -t ed25519 -C "ovh-vps"
```

Entrez une passphrase (recommandée mais non obligatoire), par exemple : `OVH2026`

Copiez la clé publique sur le VPS :

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ubuntu@164.132.43.250
```

### 5. Configuration SSH - Désactivation de l'authentification par mot de passe

Sur le VPS, éditez la configuration SSH :

```bash
sudo nano /etc/ssh/sshd_config
```

Remplacez `PasswordAuthentication yes` par `PasswordAuthentication no`

Faites de même dans le fichier cloud-init :

```bash
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
```

Redémarrez le service SSH :

```bash
sudo systemctl restart ssh
```

Vérifiez que l'authentification par mot de passe est bien désactivée (ouvrez un autre terminal) :

```bash
ssh -o PreferredAuthentications=password ubuntu@164.132.43.250
```

La connexion doit échouer.

### 6. Modification du port SSH

Modifiez le port SSH par défaut (22 → 2222) :

```bash
sudo nano /etc/ssh/sshd_config
```

Décommentez et modifiez la ligne :

```bash
Port 2222
```

Configurez le socket systemd pour le nouveau port :

```bash
sudo mkdir -p /etc/systemd/system/ssh.socket.d
sudo nano /etc/systemd/system/ssh.socket.d/override.conf
```

Ajoutez le contenu suivant :

```ini
[Socket]
ListenStream=
ListenStream=0.0.0.0:2222
ListenStream=[::]:2222
```

Redémarrez les services SSH :

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket ssh.service
sudo ss -tlnp | grep sshd
```

Testez la connexion sur le nouveau port depuis votre machine locale :

```bash
ssh -p 2222 -i ~/.ssh/id_ed25519 ubuntu@164.132.43.250
```

### 7. Installation et configuration du pare-feu (UFW)

```bash
# Installe UFW si absent
sudo apt install ufw -y

# Règles de base : tout bloquer par défaut
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Autoriser uniquement ce qui est nécessaire
sudo ufw allow 2222/tcp    # SSH (nouveau port)
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS

# Activer
sudo ufw enable
sudo ufw status verbose
```

### 8. Installation et configuration de Fail2ban

```bash
sudo apt install fail2ban -y

sudo nano /etc/fail2ban/jail.local
```

Ajoutez le contenu suivant :

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled  = true
port     = 2222
logpath  = %(sshd_log)s
backend  = systemd
```

Démarrez le service et vérifiez :

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Vérifie que SSH est bien surveillé
sudo fail2ban-client status sshd
```

### 9. Configuration du gitignore global

Pour éviter de committer des secrets par erreur :

```bash
nano ~/.gitignore
```

Ajoutez le contenu suivant :

```gitignore
.env
*.pem
*.key
*_rsa
*_ed25519
id_*
secrets.yml
config/secrets*
```

Appliquez la configuration globalement :

```bash
git config --global core.excludesfile ~/.gitignore
```

### 10. Installation de Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
```

**Déconnectez-vous et reconnectez-vous** pour appliquer les changements de groupe.

Vérifiez l'installation :

```bash
docker --version
docker compose version
```

### 11. Authentification auprès du GitHub Container Registry

```bash
echo "TON_TOKEN_ICI" | docker login ghcr.io -u TON_LOGIN --password-stdin
```

**Note** : Le token GitHub doit avoir les permissions `read:packages` et `write:packages`.

### 12. Exposition des ports Django et FastAPI

```bash
sudo ufw allow 8000/tcp   # Django
sudo ufw allow 8080/tcp   # FastAPI
sudo ufw status
```

### 13. Installation et configuration de GitHub CLI

```bash
sudo apt install gh -y
```

Authentifiez-vous :

```bash
gh auth login
```

Pour configurer les secrets GitHub, vous pouvez envoyer la clé directement depuis un fichier :

```bash
gh secret set NOM_SECRET < ~/.ssh/github_actions
```

### 14. Création du répertoire de travail

```bash
mkdir -p ~/political-prediction
cd ~/political-prediction
```

### 15. Configuration des variables d'environnement

Créez le fichier `.env` :

```bash
nano .env
```

Ajoutez les variables suivantes (remplacez les valeurs par les vôtres) :

```env
POSTGRES_USER=votre_user_postgres
POSTGRES_PASSWORD=votre_mot_de_passe_postgres
POSTGRES_DB=predilection
DATABASE_URL=postgresql://votre_user_postgres:votre_mot_de_passe_postgres@postgres:5432/predilection
SECRET_KEY=votre_secret_key_django_aleatoire
DEBUG=false
```

**Génération d'une SECRET_KEY sécurisée** :

```bash
python3 -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

### 16. Téléchargement du docker-compose.yaml

```bash
curl -sO https://raw.githubusercontent.com/NICHIKU/political-prediction/develop/docker-compose.yaml
```

---

## 🎯 Procédure de Déploiement Initial

### 1. Lancement des services

```bash
cd ~/political-prediction
docker compose pull
docker compose up -d --remove-orphans
```

### 2. Vérification de l'état des services

```bash
docker compose ps
```

Tous les services doivent afficher "Up" ou "Exited" (pour ingest qui est un one-shot).

### 3. Exécution de l'ingestion des données

Si le service ingest ne s'est pas exécuté automatiquement :

```bash
docker compose up ingest
```

Cette étape peut prendre plusieurs minutes selon la taille des données.

### 4. Vérification des logs

```bash
# Logs de tous les services
docker compose logs

# Logs d'un service spécifique
docker compose logs fastapi
docker compose logs django
docker compose logs postgres
```

### 5. Test de l'application

Ouvrez votre navigateur et accédez à :

- **Application Django** : `http://votre_ip_vps:8000`
- **API FastAPI (Swagger)** : `http://votre_ip_vps:8080/docs`
- **API FastAPI (ReDoc)** : `http://votre_ip_vps:8080/redoc`

### 6. Configuration d'un reverse proxy (optionnel mais recommandé)

Pour utiliser Nginx comme reverse proxy avec HTTPS :

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

Configurez Nginx (`/etc/nginx/sites-available/political-prediction`) :

```nginx
server {
    listen 80;
    server_name votre_domaine.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://localhost:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Activez le site et obtenez un certificat SSL :

```bash
sudo ln -s /etc/nginx/sites-available/political-prediction /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
sudo certbot --nginx -d votre_domaine.com
```

---

## 🔄 Procédure de Publication et Déploiement d'une Nouvelle Release

### Option 1 : Déploiement Automatisé via GitHub Actions (Recommandé)

Le projet utilise des workflows GitHub Actions pour le CI/CD.

#### Étapes de configuration des secrets GitHub

Dans votre repository GitHub, allez dans `Settings > Secrets and variables > Actions` et ajoutez :

- `VPS_HOST` : Adresse IP ou domaine de votre VPS (ex: 164.132.43.250)
- `VPS_SSH_PORT` : Port SSH (configuré à 2222)
- `VPS_USER` : Utilisateur SSH sur le VPS (ubuntu ou deployer)
- `VPS_SSH_KEY` : Clé privée SSH pour accéder au VPS (contenu de ~/.ssh/id_ed25519)
- `POSTGRES_USER` : Utilisateur PostgreSQL
- `POSTGRES_PASSWORD` : Mot de passe PostgreSQL
- `POSTGRES_DB` : Nom de la base de données (predilection)
- `SECRET_KEY` : Secret key Django
- `DEBUG` : `false` pour la production

**Note** : Vous pouvez également utiliser GitHub CLI pour configurer les secrets depuis le VPS :

```bash
gh auth login
gh secret set NOM_SECRET < ~/.ssh/github_actions
```

#### Processus de déploiement

1. **Poussez vos modifications** sur la branche `develop` :

```bash
git add .
git commit -m "Description des modifications"
git push origin develop
```

2. **Le workflow GitHub Actions** s'exécute automatiquement :
   - Lint du code (flake8)
   - Exécution des tests (pytest)
   - Build des images Docker
   - Push des images sur GHCR
   - Déploiement automatique sur le VPS via SSH

3. **Surveillez le déploiement** dans l'onglet "Actions" de votre repository GitHub.

### Option 2 : Déploiement Manuel

#### 1. Build et push des images localement

```bash
# Build FastAPI
docker build -t ghcr.io/votre_username/political-prediction-fastapi:latest ./api
docker push ghcr.io/votre_username/political-prediction-fastapi:latest

# Build Django
docker build -t ghcr.io/votre_username/political-prediction-django:latest ./django_political_app
docker push ghcr.io/votre_username/political-prediction-django:latest

# Build Ingest
docker build -t ghcr.io/votre_username/political-prediction-ingest:latest -f scripts/Dockerfile .
docker push ghcr.io/votre_username/political-prediction-ingest:latest
```

#### 2. Mise à jour sur le VPS

Connectez-vous au VPS avec le port SSH configuré (2222) :

```bash
ssh -p 2222 -i ~/.ssh/id_ed25519 ubuntu@164.132.43.250
cd ~/political-prediction
```

Téléchargez la nouvelle version du docker-compose.yaml :

```bash
curl -sO https://raw.githubusercontent.com/NICHIKU/political-prediction/develop/docker-compose.yaml
```

Mettez à jour les images et redémarrez :

```bash
docker compose pull
docker compose up -d --remove-orphans
```

#### 3. Nettoyage des anciennes images (optionnel)

```bash
docker image prune -a -f
```

---

## 📊 Consultation des Logs

### Logs de tous les services

```bash
docker compose logs
```

### Logs d'un service spécifique

```bash
# FastAPI
docker compose logs -f fastapi

# Django
docker compose logs -f django

# PostgreSQL
docker compose logs -f postgres

# Ingest
docker compose logs ingest
```

L'option `-f` permet de suivre les logs en temps réel (mode follow).

### Logs avec filtre temporel

```bash
# Logs des 100 dernières lignes
docker compose logs --tail=100 fastapi

# Logs depuis une heure spécifique
docker compose logs --since="2024-01-01T00:00:00" django
```

### Accès aux logs système

```bash
# Logs Docker
journalctl -u docker.service

# Logs Nginx (si configuré)
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

### Logs persistants

Pour conserver les logs après redémarrage des conteneurs, vous pouvez configurer un logging driver dans docker-compose.yaml :

```yaml
services:
  fastapi:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

---

## 🔒 Mesures de Sécurité Mises en Œuvre

### 1. Sécurité des Conteneurs
- **Images officielles** : Utilisation d'images Docker officielles (postgres:15, python:3.12-slim)
- **Utilisateur non-root** : Les conteneurs s'exécutent avec un utilisateur non-root quand possible
- **Scan de vulnérabilités** : Intégration possible de Trivy ou Snyk pour scanner les images

### 2. Sécurité Réseau
- **Firewall UFW** : Configuration stricte avec politique deny par défaut
- **Ports exposés limités** : Seuls les ports nécessaires sont ouverts (2222 SSH, 80 HTTP, 443 HTTPS, 8000 Django, 8080 FastAPI)
- **Isolation réseau** : Services Docker isolés dans un réseau dédié
- **Port SSH modifié** : Port SSH changé de 22 à 2222 pour éviter les scans automatiques

### 3. Sécurité SSH
- **Authentification par clé uniquement** : Authentification par mot de passe désactivée
- **Clé ED25519** : Utilisation de clés ED25519 plus sécurisées que RSA
- **Passphrase** : Clé SSH protégée par une passphrase
- **Fail2ban** : Protection contre les attaques par force brute avec bannissement automatique (5 essais en 10 minutes = 1 heure de bannissement)

### 4. Sécurité des Données
- **Volumes persistants** : Données PostgreSQL stockées dans un volume Docker
- **Variables d'environnement** : Secrets stockés dans des variables d'environnement, pas dans le code
- **Gitignore global** : Configuration globale pour éviter de committer des secrets (.env, clés SSH, etc.)
- **Connexion SSL** : Recommandation d'utiliser HTTPS avec Let's Encrypt

### 5. Authentification et Autorisation
- **Django Auth** : Système d'authentification intégré à Django
- **SECRET_KEY** : Clé secrète Django générée aléatoirement et stockée de manière sécurisée
- **PostgreSQL** : Authentification par mot de passe fort
- **Utilisateur dédié** : Utilisateur `deployer` créé avec droits sudo pour les opérations de déploiement

### 6. Sécurité du Déploiement
- **SSH Key-based auth** : Authentification par clé SSH pour l'accès au VPS
- **GitHub Secrets** : Secrets stockés dans GitHub Actions, jamais exposés dans le code
- **GHCR Private** : Images Docker stockées dans un registry privé (GitHub Container Registry)
- **GitHub CLI** : Utilisation de `gh` pour la configuration sécurisée des secrets

### 7. Bonnes Pratiques
- **Mode DEBUG désactivé** : `DEBUG=false` en production
- **Gunicorn workers** : 3 workers pour une meilleure résilience
- **Healthchecks** : Healthcheck configuré pour PostgreSQL
- **Whitenoise** : Servir les fichiers statiques de manière sécurisée

### Recommandations Supplémentaires

Pour renforcer davantage la sécurité :

1. **Configuration de SELinux ou AppArmor** pour renforcer la sécurité des conteneurs
2. **Rotation régulière des secrets** (mots de passe, clés API)
3. **Sauvegardes automatiques** de la base de données
4. **Monitoring** avec Prometheus/Grafana ou Uptime Robot
5. **Mise à jour régulière** des images Docker et du système
6. **Mise en place d'un WAF** (Web Application Firewall) comme ModSecurity

---

## ⚠️ Limites de la Solution

### 1. Scalabilité
- **Architecture monobase de données** : Une seule instance PostgreSQL, pas de clustering
- **Pas de load balancer** : Django et FastAPI s'exécutent sur un seul conteneur chacun
- **Workers limités** : Gunicorn configuré avec 3 workers seulement

**Impact** : La solution peut supporter une charge modérée mais n'est pas conçue pour un trafic très élevé.

### 2. Haute Disponibilité
- **Pas de redondance** : Si le VPS tombe, l'application est indisponible
- **Pas de failover** : Aucun mécanisme de basculement automatique
- **Single point of failure** : Le VPS est un SPOF (Single Point of Failure)

**Impact** : L'application a une disponibilité limitée à celle du VPS.

### 3. Sauvegardes
- **Pas de sauvegardes automatisées** : Les données ne sont sauvegardées que manuellement
- **Pas de backup distant** : Les données résident uniquement sur le VPS
- **Volume Docker local** : Si le volume est supprimé, les données sont perdues

**Impact** : Risque de perte de données en cas de défaillance matérielle ou d'erreur humaine.

### 4. Monitoring et Alerting
- **Pas de monitoring avancé** : Aucun système de monitoring intégré (Prometheus, Grafana)
- **Pas d'alerting** : Pas d'alertes automatiques en cas de problème
- **Logs limités** : Logs uniquement accessibles via Docker Compose

**Impact** : Détection tardive des problèmes et difficulté de diagnostic.

### 5. Performance
- **ML models chargés en mémoire** : Les modèles XGBoost sont chargés au démarrage
- **Pas de cache** : Aucun système de cache (Redis, Memcached)
- **Base de données non optimisée** : Pas de tuning PostgreSQL avancé

**Impact** : Temps de réponse peut être élevé pour les requêtes complexes.

### 6. Sécurité
- **Pas de WAF** : Absence de Web Application Firewall
- **Pas de rate limiting** : Aucune limitation du nombre de requêtes
- **HTTPS optionnel** : La configuration HTTPS est manuelle et non obligatoire

**Impact** : Vulnérabilité potentielle aux attaques DDoS et injections.

### 7. Maintenance
- **Déploiement manuel possible** : Bien que l'automatisation existe, le déploiement manuel reste complexe
- **Rollback manuel** : Pas de mécanisme de rollback automatique
- **Versionning des données** : Pas de migration automatique du schéma de base de données

**Impact** : Risques accrus lors des mises à jour.

### 8. Données
- **Données statiques** : Les données électorales ne se mettent pas à jour automatiquement
- **Pas de pipeline de données** : L'ingestion est manuelle ou one-shot
- **Modèle ML figé** : Le modèle de ML n'est pas retrainé automatiquement

**Impact** : Les prédictions peuvent devenir obsolètes au fil du temps.

---

## 📝 Conclusion

Cette solution de déploiement offre une base solide pour héberger l'application Predil'ection sur un VPS. Elle utilise des technologies modernes (Docker, FastAPI, Django) et intègre des bonnes pratiques de sécurité.

Cependant, pour une utilisation en production avec des exigences élevées de disponibilité, de scalabilité et de sécurité, il serait recommandé d'envisager :
- Une architecture multi-VPS avec load balancing
- Un système de sauvegardes automatisées
- Un monitoring et alerting avancés
- Un pipeline CI/CD plus robuste avec tests d'intégration
- Une base de données managée (AWS RDS, Cloud SQL, etc.)

---

**Dernière mise à jour** : 3 juin 2026
