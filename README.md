# OIDC: Authentik

## Documentation
https://docs.goauthentik.io/



## Local development setup

1. **Copy environment template**

   ```
   cp .env.template .env
   ```

   Update the copied `.env` with your secrets. Set `TRAEFIK_PORT` if you need a different local HTTP port (default `8123`).

2. **Configure hosts file**

   Add the following entries to `/etc/hosts` (Linux/macOS) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

   ```
   127.0.0.1 auth.bawes.localhost
   127.0.0.1 traefik.bawes.localhost
   ```

3. **Start the stack**

   ```bash
   docker compose pull
   docker compose up -d
   ```

4. **Access services**

   - Authentik: `http://auth.bawes.localhost:8123`
   - Traefik dashboard: `http://traefik.bawes.localhost:8123`

   > Traefik listens on `TRAEFIK_PORT` (defaults to 8123) so multiple Traefik instances can run on the same machine without conflicts.

5. **Initial authentik setup**

   Visit `http://auth.bawes.localhost:8123/if/flow/initial-setup/` and follow the onboarding flow to create the admin user.

6. **Verify configuration**

   ```
   docker compose run --rm worker dump_config
   ```

## Production deployment

1. Ensure DNS records for `auth.bawes.net` point to your server.
2. Run the production stack with SSL via Let's Encrypt:

   ```bash
   docker compose -f docker-compose.yml -f docker-compose.prod.yml pull
   docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
   ```

3. Access authentik at `https://auth.bawes.net`. The Traefik dashboard remains disabled in production.

Certificates are automatically issued and stored in `./letsencrypt` (created on first run). Traefik listens on ports 80/443 in production to satisfy Let's Encrypt HTTP-01 validation and redirect HTTP → HTTPS.