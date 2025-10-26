# Minimal Mautic via Docker Compose

This folder contains `docker-compose.yml`, a two-container stack (Mautic + MariaDB) sized for Coolify.

## Requirements
- Coolify service based on the provided compose file.
- Environment variables for the Mautic and MariaDB credentials.
- Persistent volumes (`mautic-app-data`, `mautic-shared-plugins`, `mautic-shared-themes`, `mautic-shared-media`, `mautic-db-data`).

## Deploying
1. Paste the compose file into a Coolify Docker Compose service.
2. Provide the environment variables in Coolify (or `.env`).
3. Deploy and wait until both containers report healthy.

## Uploading plugins via Coolify terminal
1. In Coolify open the Mautic service and launch the Terminal.
2. Upload the plugin bundle to the host (SCP/SFTP, or Coolify file upload if available).
3. Copy the plugin into the container:
   ```bash
   docker cp /path/to/plugin mautic:/var/www/html/plugins/
   docker exec mautic chown -R www-data:www-data /var/www/html/plugins/
   ```
4. Clear the cache inside the container:
   ```bash
   docker exec mautic php bin/console cache:clear
   ```
5. In the Mautic UI open `Settings → Plugins` and click “Install/Upgrade Plugins”.

## Backups
Use Coolify backups or `docker run --rm -v volume:/data ...` jobs to archive the named volumes regularly.
