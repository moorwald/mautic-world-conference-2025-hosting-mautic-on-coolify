# Mautic Plugin Manager with Filebrowser

This folder provides a two-stack setup: `docker-compose.mautic.yml` runs Mautic with MariaDB and persistent volumes, while `docker-compose.filebrowser.yml` starts a Filebrowser container on demand so you can upload plugins, themes, and media.

## Prerequisites
- Coolify project (or another Docker host) that can deploy both compose files.
- Environment variables matching the examples below.
- Mautic stack deployed first so the shared volumes exist.

Check the volumes after deploying Mautic in the Coolify UI.

## Mautic service (`docker-compose.mautic.yml`)
```yaml
services:
  mautic:
    image: mautic/mautic:${MAUTIC_VERSION:-5.2.8}
    container_name: mautic
    restart: unless-stopped
    environment:
      - MAUTIC_DB_HOST=db
      - MAUTIC_DB_PORT=3306
      - MAUTIC_DB_NAME=${MAUTIC_DB_NAME:-mautic}
      - MAUTIC_DB_USER=${MAUTIC_DB_USER:-mautic}
      - MAUTIC_DB_PASSWORD=${MAUTIC_DB_PASSWORD}
      - MAUTIC_TRUSTED_HOSTS=${MAUTIC_TRUSTED_HOSTS}
      - MAUTIC_RUN_CRON_JOBS=true
      - PHP_MEMORY_LIMIT=${PHP_MEMORY_LIMIT:-512M}
      - PHP_MAX_UPLOAD=${PHP_MAX_UPLOAD:-256M}
      - PHP_MAX_EXECUTION_TIME=${PHP_MAX_EXECUTION_TIME:-300}
    volumes:
      - mautic_data:/var/www/html
      - mautic_plugins:/var/www/html/plugins
      - mautic_themes:/var/www/html/themes
      - mautic_media:/var/www/html/media
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    networks:
      - mautic-network

  db:
    image: mariadb:${MARIADB_VERSION:-10.11}
    container_name: mautic-db
    restart: unless-stopped
    environment:
      - MARIADB_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MARIADB_DATABASE=${MAUTIC_DB_NAME:-mautic}
      - MARIADB_USER=${MAUTIC_DB_USER:-mautic}
      - MARIADB_PASSWORD=${MAUTIC_DB_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - mautic-network
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci

volumes:
  mautic_data:
    name: mautic-app-data
    driver: local
  mautic_plugins:
    name: mautic-shared-plugins
    driver: local
  mautic_themes:
    name: mautic-shared-themes
    driver: local
  mautic_media:
    name: mautic-shared-media
    driver: local
  db_data:
    name: mautic-db-data
    driver: local

networks:
  mautic-network:
    name: mautic-network
    driver: bridge
```

## Filebrowser service (`docker-compose.filebrowser.yml`)
```yaml
services:
  filebrowser:
    image: filebrowser/filebrowser:latest
    container_name: mautic-plugin-manager
    restart: "no"
    user: "33:33"
    environment:
      - FB_NOAUTH=false
      - FB_AUTH_METHOD=json
      - FB_USERNAME=${FB_ADMIN_USER:-admin}
      - FB_PASSWORD=${FB_ADMIN_PASSWORD:?FB_ADMIN_PASSWORD is required}
      - FB_ROOT=/srv/mautic
      - FB_DATABASE=/config/database.db
      - FB_LOG=stdout
      - FB_BASEURL=
    volumes:
      - mautic_plugins:/srv/mautic/plugins
      - mautic_themes:/srv/mautic/themes
      - mautic_media:/srv/mautic/media
      - filebrowser_config:/config
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

volumes:
  mautic_plugins:
    name: mautic-shared-plugins
    external: true
  mautic_themes:
    name: mautic-shared-themes
    external: true
  mautic_media:
    name: mautic-shared-media
    external: true
  filebrowser_config:
    name: filebrowser-config
    driver: local
```

## Environment variables
```env
MAUTIC_VERSION=5.2.8
MAUTIC_DB_NAME=mautic
MAUTIC_DB_USER=mautic
MAUTIC_DB_PASSWORD=change-me
MYSQL_ROOT_PASSWORD=change-me-too
MAUTIC_TRUSTED_HOSTS=mautic.example.com
FB_ADMIN_USER=admin
FB_ADMIN_PASSWORD=generate-a-long-random-secret
```

## Deploy sequence
- Deploy the Mautic compose file in Coolify and wait for both services to become healthy.
- Confirm the shared volumes exist with `docker volume ls`.
- Add a second Docker Compose service in Coolify, paste the Filebrowser compose file, and disable auto-redeploy so the container stays stopped until needed.

## Upload workflow
- Start the Filebrowser service only when you want to manage files.
- Log in using `FB_ADMIN_USER`/`FB_ADMIN_PASSWORD`.
- Upload plugin archives to `/srv/mautic/plugins/`, themes to `/srv/mautic/themes/`, and media to `/srv/mautic/media/`.
- Extract archives as needed and stop the service again once uploads are complete.
- Inside the running Mautic container clear the cache: `php bin/console cache:clear`.
- In the Mautic UI go to `Settings → Plugins` and run “Install/Upgrade Plugins”.

## Security
- Keep Filebrowser stopped by default.
- Use a long, unique `FB_ADMIN_PASSWORD` and restrict access with Coolify’s networking features when possible.
- Redeploy periodically to pick up upstream image updates.
