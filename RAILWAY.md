# Railway Deployment Guide

This guide explains how to deploy Authentik on Railway using Dockerfiles and Railway's managed PostgreSQL service.

## Overview

Railway requires **one Dockerfile per service** (not docker-compose). Authentik has two services:
- **Server** - Handles HTTP requests (uses `Dockerfile.server`)
- **Worker** - Handles background tasks (uses `Dockerfile.worker`)

**Key Features:**
- ✅ **No Traefik needed** - Railway automatically handles SSL/TLS termination with Let's Encrypt certificates
- ✅ **Automatic HTTPS** - Railway issues and renews SSL certificates automatically for custom domains
- ✅ **Managed PostgreSQL** - Railway provides a fully managed database service
- ✅ **Simplified deployment** - No reverse proxy configuration required

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

You **must** set these environment variables in Railway's dashboard for **both** server and worker services:

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

**Important:** This must be a secure, random string. Never commit this to your repository. Use the **same value** for both server and worker services.

#### 2. Database Connection Variables (Required)

Set these for both server and worker services to connect to Railway's PostgreSQL:

```
AUTHENTIK_POSTGRESQL__HOST=${PGHOST}
AUTHENTIK_POSTGRESQL__PORT=${PGPORT}
AUTHENTIK_POSTGRESQL__NAME=${PGDATABASE}
AUTHENTIK_POSTGRESQL__USER=${PGUSER}
AUTHENTIK_POSTGRESQL__PASSWORD=${PGPASSWORD}
```

**Note:** Railway provides `PGHOST`, `PGPORT`, etc. automatically, but Authentik needs them in the `AUTHENTIK_POSTGRESQL__*` format. You can reference Railway's variables using `${PGHOST}` syntax.

## Deployment Steps

### Step 1: Create Railway Project and PostgreSQL

1. Create a new Railway project
2. Add a PostgreSQL service (Railway will automatically provide database connection variables)

### Step 2: Deploy Server Service

1. **Add a new service** in Railway:
   - Click "New" → "GitHub Repo" (or "Empty Service")
   - Connect your repository

2. **Configure the service:**
   - Go to Settings → Dockerfile Path
   - Set Dockerfile path to: `Dockerfile.server`
   - Set the port to: `9000` (in Settings → Networking)

3. **Set environment variables** (Settings → Variables):
   ```
   AUTHENTIK_SECRET_KEY=<your-generated-secret-key>
   AUTHENTIK_POSTGRESQL__HOST=${PGHOST}
   AUTHENTIK_POSTGRESQL__PORT=${PGPORT}
   AUTHENTIK_POSTGRESQL__NAME=${PGDATABASE}
   AUTHENTIK_POSTGRESQL__USER=${PGUSER}
   AUTHENTIK_POSTGRESQL__PASSWORD=${PGPASSWORD}
   ```

4. **Deploy** - Railway will automatically build and deploy using `Dockerfile.server`

### Step 3: Deploy Worker Service

1. **Add another service** in the same Railway project:
   - Click "New" → "GitHub Repo" (select the same repository)
   - Or duplicate the server service and modify it

2. **Configure the service:**
   - Go to Settings → Dockerfile Path
   - Set Dockerfile path to: `Dockerfile.worker`
   - Worker doesn't need a public port, but Railway may require one for health checks

3. **Set environment variables** (Settings → Variables):
   ```
   AUTHENTIK_SECRET_KEY=<same-secret-key-as-server>
   AUTHENTIK_POSTGRESQL__HOST=${PGHOST}
   AUTHENTIK_POSTGRESQL__PORT=${PGPORT}
   AUTHENTIK_POSTGRESQL__NAME=${PGDATABASE}
   AUTHENTIK_POSTGRESQL__USER=${PGUSER}
   AUTHENTIK_POSTGRESQL__PASSWORD=${PGPASSWORD}
   ```

4. **Deploy** - Railway will automatically build and deploy using `Dockerfile.worker`

### Step 4: Configure Custom Domain (Optional but Recommended)

1. **For the Server service:**
   - Go to Settings → Networking
   - Add your custom domain (e.g., `auth.yourdomain.com`)
   - Railway will automatically:
     - Issue a Let's Encrypt SSL certificate
     - Configure HTTPS redirects
     - Handle certificate renewals
   - Update your DNS to point to Railway's provided CNAME

### Step 5: Initial Setup

1. Visit your Railway-deployed Authentik URL (either Railway's default domain or your custom domain)
2. Navigate to `/if/flow/initial-setup/` to create the admin user

## SSL/TLS Configuration

Railway automatically handles SSL/TLS termination, so **Traefik is not needed**:

- **Automatic HTTPS**: Railway issues Let's Encrypt certificates automatically
- **Certificate Renewal**: Certificates are renewed automatically before expiration
- **HTTP to HTTPS Redirect**: Railway automatically redirects HTTP traffic to HTTPS
- **Custom Domains**: Simply add your domain in Railway's dashboard and configure DNS

The Dockerfiles:
- `Dockerfile.server` - Exposes port 9000 for Railway to route traffic
- `Dockerfile.worker` - Background service, no public HTTP endpoint needed

## Project Structure

```
.
├── Dockerfile.server      # Server service Dockerfile
├── Dockerfile.worker      # Worker service Dockerfile
├── docker-compose.yml     # For local development
├── docker-compose.prod.yml # For self-hosted production
└── RAILWAY.md            # This file
```

## Security Notes

- ✅ **Never commit** `AUTHENTIK_SECRET_KEY` to your repository
- ✅ Railway automatically manages PostgreSQL credentials securely
- ✅ Railway automatically manages SSL certificates securely
- ✅ All sensitive data is stored in Railway's environment variables, not in code
- ✅ Use Railway's "Sealed Variables" feature for extra-sensitive data if needed
- ✅ Use the **same** `AUTHENTIK_SECRET_KEY` for both server and worker services

## Troubleshooting

### Database Connection Issues

If Authentik can't connect to the database:
1. Verify PostgreSQL service is running in Railway
2. Check that `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, and `PGPASSWORD` are set in Railway
3. Verify that `AUTHENTIK_POSTGRESQL__*` variables are correctly set in both services
4. Ensure all services are in the same Railway project (for private networking)

### Service Won't Start

- Check Railway logs for errors
- Verify `AUTHENTIK_SECRET_KEY` is set in both services
- Ensure all required environment variables are present
- Verify Dockerfile path is correct in Railway settings

### Port Configuration

- **Server**: Must expose port `9000` (configure in Railway service settings)
- **Worker**: Doesn't need a public port, but Railway may require one for health checks

## Environment Variable Reference

| Variable | Source | Required | Description |
|----------|--------|----------|-------------|
| `PGHOST` | Railway (auto) | Yes | PostgreSQL hostname |
| `PGPORT` | Railway (auto) | Yes | PostgreSQL port (default: 5432) |
| `PGDATABASE` | Railway (auto) | Yes | Database name |
| `PGUSER` | Railway (auto) | Yes | PostgreSQL username |
| `PGPASSWORD` | Railway (auto) | Yes | PostgreSQL password |
| `AUTHENTIK_SECRET_KEY` | Manual | **Yes** | Secret key for Authentik encryption (same for both services) |
| `AUTHENTIK_POSTGRESQL__HOST` | Manual | **Yes** | Set to `${PGHOST}` to reference Railway's variable |
| `AUTHENTIK_POSTGRESQL__PORT` | Manual | **Yes** | Set to `${PGPORT}` to reference Railway's variable |
| `AUTHENTIK_POSTGRESQL__NAME` | Manual | **Yes** | Set to `${PGDATABASE}` to reference Railway's variable |
| `AUTHENTIK_POSTGRESQL__USER` | Manual | **Yes** | Set to `${PGUSER}` to reference Railway's variable |
| `AUTHENTIK_POSTGRESQL__PASSWORD` | Manual | **Yes** | Set to `${PGPASSWORD}` to reference Railway's variable |

## Quick Reference

**Server Service:**
- Dockerfile: `Dockerfile.server`
- Port: `9000`
- Required env vars: `AUTHENTIK_SECRET_KEY`, `AUTHENTIK_POSTGRESQL__*`

**Worker Service:**
- Dockerfile: `Dockerfile.worker`
- Port: Not required (background service)
- Required env vars: `AUTHENTIK_SECRET_KEY`, `AUTHENTIK_POSTGRESQL__*` (same as server)
