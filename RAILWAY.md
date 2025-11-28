# Railway Deployment Guide

This guide explains how to deploy Authentik on Railway using Railway's managed PostgreSQL service.

## Overview

Railway provides a managed PostgreSQL instance, so we don't need to run PostgreSQL in Docker. The `docker-compose.railway.yml` file replaces the PostgreSQL service with a minimal placeholder (to satisfy service dependencies) and configures Authentik to use Railway's database via environment variables.

**Key Features:**
- ✅ **No Traefik needed** - Railway automatically handles SSL/TLS termination with Let's Encrypt certificates
- ✅ **Automatic HTTPS** - Railway issues and renews SSL certificates automatically for custom domains
- ✅ **Managed PostgreSQL** - Railway provides a fully managed database service
- ✅ **Simplified deployment** - No reverse proxy configuration required

**Note:** The postgresql service in the Railway compose file is a placeholder that satisfies Docker Compose dependencies. Authentik will connect to Railway's managed Postgres using the environment variables, not the placeholder service. Traefik is disabled via profiles since Railway handles SSL/TLS automatically.

## Required Environment Variables

### Automatically Provided by Railway (PostgreSQL Service)

When you add a PostgreSQL service to your Railway project, Railway automatically provides these environment variables. **You don't need to set these manually:**

- `PGHOST` - PostgreSQL hostname (e.g., `containers-us-west-xxx.railway.app`)
- `PGPORT` - PostgreSQL port (typically `5432`)
- `PGDATABASE` - Database name (e.g., `railway`)
- `PGUSER` - PostgreSQL username (e.g., `postgres`)
- `PGPASSWORD` - PostgreSQL password
- `DATABASE_URL` - Full connection string (public URL)
- `DATABASE_PUBLIC_URL` - Public connection string
- `RAILWAY_PRIVATE_DOMAIN` - Private domain for database connection
- `RAILWAY_TCP_PROXY_DOMAIN` - TCP proxy domain
- `RAILWAY_TCP_PROXY_PORT` - TCP proxy port

### Required Manual Configuration

You **must** set these environment variables in Railway's dashboard:

#### 1. Authentik Secret Key (Required)
```
AUTHENTIK_SECRET_KEY=<generate-a-secure-random-string>
```

**How to generate:**
```bash
# Option 1: Using openssl
openssl rand -base64 32

# Option 2: Using Python
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

**Important:** This must be a secure, random string. Never commit this to your repository.

#### 2. Authentik Image & Tag (Optional)
```
AUTHENTIK_IMAGE=ghcr.io/goauthentik/server
AUTHENTIK_TAG=2025.10.2
```

These default to the values in `docker-compose.yml` if not set.

### Optional Environment Variables

- `AUTHENTIK_IMAGE` - Authentik Docker image (defaults to `ghcr.io/goauthentik/server`)
- `AUTHENTIK_TAG` - Authentik Docker tag (defaults to `2025.10.2`)

## Deployment Steps

1. **Create a Railway project** and add a PostgreSQL service

2. **Set required environment variables** in Railway:
   - Go to your project → Variables
   - Add `AUTHENTIK_SECRET_KEY` with a secure random value
   - Optionally set `AUTHENTIK_IMAGE` and `AUTHENTIK_TAG` if you want specific versions

3. **Deploy using the Railway override:**
   
   Railway will automatically use `docker-compose.railway.yml` when detected, or you can specify it:
   
   ```bash
   docker compose -f docker-compose.yml -f docker-compose.railway.yml up -d
   ```

4. **Configure Railway service:**
   - In Railway dashboard, go to your service settings
   - Set the service to expose port `9000` (Authentik server port)
   - The `docker-compose.railway.yml` already configures this, but verify in Railway's service settings

5. **Configure custom domain (optional but recommended):**
   - In Railway dashboard, go to your service → Settings → Networking
   - Add your custom domain (e.g., `auth.yourdomain.com`)
   - Railway will automatically:
     - Issue a Let's Encrypt SSL certificate
     - Configure HTTPS redirects
     - Handle certificate renewals
   - Update your DNS to point to Railway's provided CNAME

6. **Initial setup:**
   - Visit your Railway-deployed Authentik URL (either Railway's default domain or your custom domain)
   - Navigate to `/if/flow/initial-setup/` to create the admin user

## SSL/TLS Configuration

Railway automatically handles SSL/TLS termination, so **Traefik is not needed**:

- **Automatic HTTPS**: Railway issues Let's Encrypt certificates automatically
- **Certificate Renewal**: Certificates are renewed automatically before expiration
- **HTTP to HTTPS Redirect**: Railway automatically redirects HTTP traffic to HTTPS
- **Custom Domains**: Simply add your domain in Railway's dashboard and configure DNS

The `docker-compose.railway.yml` file:
- Disables Traefik service (via profiles)
- Removes Traefik labels from the server service
- Exposes port 9000 directly for Railway to route traffic

## Security Notes

- ✅ **Never commit** `AUTHENTIK_SECRET_KEY` to your repository
- ✅ Railway automatically manages PostgreSQL credentials securely
- ✅ Railway automatically manages SSL certificates securely
- ✅ All sensitive data is stored in Railway's environment variables, not in code
- ✅ Use Railway's "Sealed Variables" feature for extra-sensitive data if needed

## Troubleshooting

### Database Connection Issues

If Authentik can't connect to the database:
1. Verify PostgreSQL service is running in Railway
2. Check that `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, and `PGPASSWORD` are set
3. Ensure the services are in the same Railway project (for private networking)

### Service Won't Start

- Check Railway logs for errors
- Verify `AUTHENTIK_SECRET_KEY` is set
- Ensure all required environment variables are present

## Environment Variable Reference

| Variable | Source | Required | Description |
|----------|--------|----------|-------------|
| `PGHOST` | Railway (auto) | Yes | PostgreSQL hostname |
| `PGPORT` | Railway (auto) | Yes | PostgreSQL port (default: 5432) |
| `PGDATABASE` | Railway (auto) | Yes | Database name |
| `PGUSER` | Railway (auto) | Yes | PostgreSQL username |
| `PGPASSWORD` | Railway (auto) | Yes | PostgreSQL password |
| `AUTHENTIK_SECRET_KEY` | Manual | **Yes** | Secret key for Authentik encryption |
| `AUTHENTIK_IMAGE` | Manual | No | Authentik Docker image (default: ghcr.io/goauthentik/server) |
| `AUTHENTIK_TAG` | Manual | No | Authentik Docker tag (default: 2025.10.2) |

