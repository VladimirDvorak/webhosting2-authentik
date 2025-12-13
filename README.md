# Authentik Identity Provider

Standalone Authentik deployment for the webhosting2 project. This repository contains the Docker Compose configuration for running Authentik as an identity provider that can be used by multiple services (e.g., BFF instances).

## Overview

This is a separate, independently maintained Authentik instance based on the [official Authentik Docker Compose setup](https://goauthentik.io/docs/installation/docker-compose). It provides OAuth2/OIDC authentication services for the webhosting2 ecosystem.

## Architecture

```
┌─────────────┐         ┌─────────────┐
│  BFF App 1  │ ──────> │             │
└─────────────┘         │  Authentik  │
                        │  (This Repo)│
┌─────────────┐         │             │
│  BFF App 2  │ ──────> │             │
└─────────────┘         └─────────────┘
```

Multiple BFF instances can connect to this single Authentik instance, each with their own OAuth2 client configuration.

## Quick Start

### Prerequisites

- Docker and Docker Compose
- At least 2GB of available RAM
- Ports 9000 (HTTP) and 9443 (HTTPS) available

### 1. Clone and Setup

```bash
cd webhosting2-authentik
cp .env.example .env
# Edit .env with your secure passwords and secret key
```

### 2. Generate Secure Values

**Generate a secure secret key:**
```bash
# Using openssl
openssl rand -base64 32

# Or using Python
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

**Generate a secure database password:**
```bash
openssl rand -base64 24
```

### 3. Configure Environment Variables

Edit `.env` file:

```env
AUTHENTIK_DB_NAME=authentik
AUTHENTIK_DB_USER=authentik
AUTHENTIK_DB_PASSWORD=<your-secure-password>
AUTHENTIK_SECRET_KEY=<your-secure-secret-key>
AUTHENTIK_PORT_HTTP=9000
AUTHENTIK_PORT_HTTPS=9443
```

### 4. Start Authentik

```bash
docker-compose up -d
```

This will start:
- **PostgreSQL** (database for Authentik)
- **Authentik Server** (main application on port 9000)
- **Authentik Worker** (background tasks)

### 5. Initial Setup

1. Wait for services to be ready (check logs: `docker-compose logs -f server`)
2. Access Authentik admin panel: `http://localhost:9000/if/admin/`
3. Complete the initial setup wizard:
   - Create an admin user
   - Set up your organization details

### 6. Verify Installation

Check that all services are running:

```bash
docker-compose ps
```

All services should show "Up" status.

## Configuration

### Port Configuration

Default ports:
- **HTTP**: 9000
- **HTTPS**: 9443

To change ports, update `.env`:

```env
AUTHENTIK_PORT_HTTP=9001
AUTHENTIK_PORT_HTTPS=9444
```

### Database Configuration

The default PostgreSQL configuration uses:
- Database: `authentik`
- User: `authentik`
- Password: Set in `.env` as `AUTHENTIK_DB_PASSWORD`

### Image Version

To use a specific Authentik version:

```env
AUTHENTIK_TAG=2025.10.2
```

Or use a different image:

```env
AUTHENTIK_IMAGE=ghcr.io/goauthentik/server
AUTHENTIK_TAG=2025.10.2
```

## Setting Up OAuth2 Clients for BFF

After Authentik is running, you need to configure OAuth2 providers for each BFF instance:

### 1. Create OAuth2/OpenID Provider

1. Go to **Applications** → **Providers**
2. Click **Create** → **OAuth2/OpenID Provider**
3. Configure:
   - **Name**: `BFF Provider - Instance 1` (or unique name)
   - **Client type**: `Confidential`
   - **Client ID**: Must match `AUTHENTIK_CLIENT_ID` in your BFF's `.env`
   - **Client secret**: Generate and copy to BFF's `.env` as `AUTHENTIK_CLIENT_SECRET`
   - **Redirect URIs**: Must match your BFF's redirect URI
     - Example: `http://localhost:5000/auth/callback` (for BFF on port 5000)
   - **Scopes**: `openid`, `email`, `profile`
   - **Sub mode**: `user_email` or `user_username`

### 2. Create Application

1. Go to **Applications** → **Applications**
2. Click **Create**
3. Configure:
   - **Name**: `BFF Application - Instance 1` (or unique name)
   - **Slug**: `bff-app-1` (or unique slug)
   - **Provider**: Select the provider created above
   - **Launch URL**: Your BFF URL (e.g., `http://localhost:5000`)

### 3. Assign Users

1. Go to **Directory** → **Users**
2. Create users or assign existing users to the application
3. Users can now authenticate via the BFF

## Multiple BFF Instances

You can create multiple OAuth2 providers and applications for different BFF instances:

- Each BFF instance gets its own provider with unique:
  - Client ID
  - Client Secret
  - Redirect URI (matching the BFF's port)

Example:
- BFF Instance 1: Port 5000 → Client ID: `bff-client-1`
- BFF Instance 2: Port 5001 → Client ID: `bff-client-2`

## Maintenance

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f server
docker-compose logs -f worker
docker-compose logs -f postgresql
```

### Stop Services

```bash
docker-compose stop
```

### Start Services

```bash
docker-compose start
```

### Restart Services

```bash
docker-compose restart
```

### Update Authentik

1. Update `AUTHENTIK_TAG` in `.env` to the desired version
2. Pull new images: `docker-compose pull`
3. Restart services: `docker-compose up -d`

### Backup Database

```bash
# Create backup
docker-compose exec postgresql pg_dump -U authentik authentik > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore backup
docker-compose exec -T postgresql psql -U authentik authentik < backup_20241213_120000.sql
```

## Volumes

The following directories are created and persisted:

- `./media/` - Authentik media files
- `./certs/` - SSL certificates (if using HTTPS)
- `./custom-templates/` - Custom email/template overrides
- `database/` (Docker volume) - PostgreSQL data

## Security Considerations

1. **Secret Key**: Always use a strong, random `AUTHENTIK_SECRET_KEY`
2. **Database Password**: Use a strong password for `AUTHENTIK_DB_PASSWORD`
3. **HTTPS**: In production, configure HTTPS and use proper SSL certificates
4. **Firewall**: Restrict access to ports 9000/9443 in production
5. **Backups**: Regularly backup the PostgreSQL database
6. **Updates**: Keep Authentik updated to the latest stable version

## Production Deployment

For production:

1. **Use HTTPS**: Configure proper SSL certificates in `./certs/`
2. **Strong Secrets**: Generate secure random values for all secrets
3. **Network Security**: Use a reverse proxy (nginx/traefik) in front
4. **Monitoring**: Set up health checks and monitoring
5. **Backups**: Implement automated database backups
6. **Resource Limits**: Set appropriate CPU/memory limits in docker-compose

Example production `.env` additions:

```env
# Use environment-specific values
AUTHENTIK_DB_PASSWORD=<production-secure-password>
AUTHENTIK_SECRET_KEY=<production-secure-secret>
AUTHENTIK_TAG=2025.10.2  # Pin to specific version
```

## Troubleshooting

### Services Won't Start

Check logs:
```bash
docker-compose logs
```

Common issues:
- Port already in use: Change `AUTHENTIK_PORT_HTTP` in `.env`
- Database connection failed: Verify `AUTHENTIK_DB_PASSWORD` is set correctly
- Secret key missing: Ensure `AUTHENTIK_SECRET_KEY` is set

### Can't Access Admin Panel

1. Wait for services to fully start (may take 1-2 minutes)
2. Check server logs: `docker-compose logs server`
3. Verify port is accessible: `curl http://localhost:9000/if/admin/`

### Database Connection Errors

1. Verify PostgreSQL is healthy: `docker-compose ps postgresql`
2. Check database credentials in `.env`
3. Restart services: `docker-compose restart`

## Related Projects

- **webhosting2-bff**: Backend for Frontend that connects to this Authentik instance

## Resources

- [Authentik Documentation](https://goauthentik.io/docs/)
- [Authentik Docker Compose Guide](https://goauthentik.io/docs/installation/docker-compose)
- [OAuth2/OIDC Configuration](https://goauthentik.io/docs/providers/oauth2/)

## License

This configuration follows the same license as Authentik itself. See [Authentik License](https://github.com/goauthentik/authentik/blob/main/LICENSE) for details.

