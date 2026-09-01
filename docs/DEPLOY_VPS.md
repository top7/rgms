# Déploiement MediaCMS sur le VPS (media.romain-giusiano.com)

Ce dépôt est un fork de MediaCMS utilisé comme plateforme vidéo privée
pour le site https://romain-giusiano.com (sections Film et CG Art).
Il remplace les lecteurs Vimeo par un lecteur auto-hébergé.

## Architecture

```
Internet -> Nginx hôte (72.62.186.172)
              ├── romain-giusiano.com        -> 127.0.0.1:3001 (site Next.js)
              ├── romain-giusiano.com/directus/ -> 127.0.0.1:8055 (Directus CMS)
              └── media.romain-giusiano.com  -> 127.0.0.1:9000 (MediaCMS/Docker)
```

MediaCMS tourne dans Docker avec : web (nginx+uwsgi), migrations, celery_beat,
celery_worker, postgres, redis. Seul le port 9000 est exposé, en local.

## Installation

### 1. DNS (Hostinger)

Ajouter un enregistrement A :

| Type | Nom  | Pointe vers    | TTL  |
|------|------|----------------|------|
| A    | media| 72.62.186.172  | 1800 |

### 2. Docker

```bash
curl -fsSL https://get.docker.com | sh
```

### 3. MediaCMS

```bash
git clone https://github.com/top7/rgms.git /var/www/mediacms
cd /var/www/mediacms

# Premier démarrage : définir le mot de passe admin
export FRONTEND_HOST=https://media.romain-giusiano.com
ADMIN_USER=admin ADMIN_EMAIL=admin@romain-giusiano.com ADMIN_PASSWORD='****' \
  docker compose -f docker-compose/docker-compose-vps.yaml up -d
```

Les démarrages suivants n'ont plus besoin des variables ADMIN_* :

```bash
docker compose -f docker-compose/docker-compose-vps.yaml up -d
```

### 4. Nginx hôte

`/etc/nginx/sites-available/media.romain-giusiano.com` :

```nginx
server {
    server_name media.romain-giusiano.com;

    client_max_body_size 10G;
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;

    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 80;
}
```

```bash
ln -s /etc/nginx/sites-available/media.romain-giusiano.com /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
certbot --nginx -d media.romain-giusiano.com
```

### 5. Configuration MediaCMS

Dans l'admin Django (`/django-admin` si DJANGO_ADMIN_URL est défini, sinon
via l'interface `/admin`) :

- Global settings : activer "Private video site" / désactiver l'inscription
  publique selon les besoins.

## Utilisation avec le site romain-giusiano.com

Après upload d'une vidéo dans MediaCMS, copier l'URL de la vidéo :

```
https://media.romain-giusiano.com/view?m=<friendly_token>
```

et la coller dans le champ "video_url" de l'admin personnalisé du site
(section Film ou CG Art). Le lecteur intégré utilise automatiquement :

```
https://media.romain-giusiano.com/embed?m=<friendly_token>
```

Les miniatures sont récupérées via l'API publique :

```
https://media.romain-giusiano.com/api/v1/media/<friendly_token>
```

## Commandes utiles

```bash
docker compose -f docker-compose/docker-compose-vps.yaml ps       # état
docker compose -f docker-compose/docker-compose-vps.yaml logs -f  # logs
docker compose -f docker-compose/docker-compose-vps.yaml restart  # redémarrer
docker compose -f docker-compose/docker-compose-vps.yaml pull     # mise à jour image
```

## Sauvegarde

```bash
# Base de données
docker compose -f docker-compose/docker-compose-vps.yaml exec db \
  pg_dump -U mediacms mediacms > backup_$(date +%F).sql

# Médias (volume media_store)
docker run --rm -v rgms_media_store:/data -v $(pwd):/backup alpine \
  tar czf /backup/media_files_$(date +%F).tar.gz -C /data .
```
