# Supabase Self-Hosted on Coolify v4

This guide covers deploying Supabase using Coolify v4.

## Prerequisites

- Coolify v4 instance running
- Domain configured (or using sslip.io)
- Git repository with this codebase

## Files Overview

| File                         | Purpose                                     |
| ---------------------------- | ------------------------------------------- |
| `docker-compose.coolify.yml` | Adapted compose file for Coolify            |
| `.env.prod`                  | Environment variables                       |
| `volumes/`                   | Configuration files mounted into containers |

## Quick Start

### 1. Create New Application in Coolify

1. Go to your Coolify dashboard
2. Create a new **Docker Compose** application
3. Connect your Git repository containing this codebase
4. Set the **Docker Compose file** to: `docker-compose.coolify.yml`

### 2. Configure Environment Variables

In Coolify's environment variables section, either:

- **Option A**: Copy all contents from `env.prod`
- **Option B**: Use Coolify's "Import from .env" feature with `env.prod`

### 3. Critical Variables to Update

Before deploying, update these values in Coolify:

```bash
# Your actual domain (update these!)
SERVICE_FQDN_KONG=your-domain.com
SUPABASE_PUBLIC_URL=https://your-domain.com
API_EXTERNAL_URL=https://your-domain.com
SITE_URL=https://your-domain.com

# Dashboard credentials (change these!)
DASHBOARD_USERNAME=admin
DASHBOARD_PASSWORD=your-very-secure-password

# SMTP for production emails
SMTP_HOST=smtp.yourmailprovider.com
SMTP_PORT=587
SMTP_USER=your-smtp-user
SMTP_PASS=your-smtp-password
SMTP_ADMIN_EMAIL=admin@yourdomain.com
```

### 4. Configure Traefik Routing

In Coolify, set the domain for your application to match `SERVICE_FQDN_KONG`.

The Kong service has Traefik labels that will:

- Route all traffic to Kong on port 8000
- Handle WebSocket connections for Realtime

### 5. Deploy

Click **Deploy** in Coolify and wait for all services to become healthy.

## Service Architecture

```
                    ┌─────────────┐
                    │   Traefik   │ (Coolify's reverse proxy)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    Kong     │ :8000 (API Gateway)
                    └──────┬──────┘
           ┌───────────────┼───────────────┬───────────────┐
           │               │               │               │
    ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
    │   Studio    │ │    Auth     │ │    REST     │ │  Realtime   │
    │   (UI)      │ │  (GoTrue)   │ │ (PostgREST) │ │ (WebSocket) │
    └─────────────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
                           │               │               │
                    ┌──────▼───────────────▼───────────────▼──────┐
                    │              PostgreSQL                      │
                    │           (supabase/postgres)                │
                    └─────────────────────────────────────────────┘
```

## URL Endpoints

After deployment, your Supabase instance will be available at:

| Endpoint  | URL                                     | Purpose                 |
| --------- | --------------------------------------- | ----------------------- |
| Dashboard | `https://your-domain.com/`              | Studio UI               |
| Auth API  | `https://your-domain.com/auth/v1/`      | Authentication          |
| REST API  | `https://your-domain.com/rest/v1/`      | Database REST API       |
| Realtime  | `wss://your-domain.com/realtime/v1/`    | WebSocket subscriptions |
| Storage   | `https://your-domain.com/storage/v1/`   | File storage            |
| Functions | `https://your-domain.com/functions/v1/` | Edge Functions          |

## Dashboard Login

The Studio dashboard is protected with Basic Auth:

- **Username**: Value of `DASHBOARD_USERNAME`
- **Password**: Value of `DASHBOARD_PASSWORD`

## Connecting Your App

Use these values in your application:

```javascript
// JavaScript/TypeScript
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  "https://your-domain.com", // SUPABASE_PUBLIC_URL
  "your-anon-key" // ANON_KEY from env.prod
);
```

```dart
// Flutter/Dart
import 'package:supabase_flutter/supabase_flutter.dart';

await Supabase.initialize(
  url: 'https://your-domain.com',
  anonKey: 'your-anon-key',
);
```

## Troubleshooting

### 1. Services Not Starting

Check the startup order. Services have health check dependencies:

```
vector → db → analytics → kong → studio
                       → auth
                       → rest
                       → realtime
                       → storage
                       → functions
                       → meta
```

### 2. Logs Not Appearing in Studio

The Vector service requires Docker socket access to collect logs. If Coolify restricts this:

- Logs won't appear in Studio's Log Explorer
- All other features will work normally
- Consider using Coolify's built-in log viewer instead

### 3. WebSocket/Realtime Not Working

Ensure Traefik is configured for WebSocket:

- The Kong service includes WebSocket middleware headers
- Check that your domain supports WebSocket connections

### 4. Database Connection Issues

If services can't connect to the database:

1. Check `POSTGRES_HOST=supabase-db` is correct
2. Verify `POSTGRES_PASSWORD` matches across all services
3. Wait for db health check to pass before other services start

### 5. Storage/MinIO Issues

MinIO runs internally. If you need external access to MinIO console:

1. Uncomment the ports in `docker-compose.coolify.yml`
2. Configure additional Traefik routing for MinIO

## Volumes & Persistence

The following data is persisted:

| Volume         | Data                     |
| -------------- | ------------------------ |
| `db-data`      | PostgreSQL database      |
| `db-config`    | PostgreSQL configuration |
| `minio-data`   | Uploaded files           |
| `storage-data` | Storage API cache        |

## Security Checklist

Before going to production:

- [ ] Change `DASHBOARD_PASSWORD` to a strong password
- [ ] Change `JWT_SECRET` and regenerate `ANON_KEY` / `SERVICE_ROLE_KEY`
- [ ] Configure real SMTP settings
- [ ] Enable HTTPS in Coolify
- [ ] Review `ADDITIONAL_REDIRECT_URLS` for OAuth
- [ ] Set `DISABLE_SIGNUP=true` if you don't want public registration

## Regenerating JWT Keys

If you need new JWT keys:

```bash
# Generate a new JWT_SECRET (min 32 characters)
openssl rand -base64 32

# Then use https://supabase.com/docs/guides/self-hosting#api-keys
# Or use jwt.io to generate ANON_KEY and SERVICE_ROLE_KEY with:
# - Header: {"alg": "HS256", "typ": "JWT"}
# - Payload for anon: {"role": "anon", "iss": "supabase", "iat": 1764441000, "exp": 1922207400}
# - Payload for service_role: {"role": "service_role", "iss": "supabase", "iat": 1764441000, "exp": 1922207400}
```

## Support

- [Supabase Self-Hosting Docs](https://supabase.com/docs/guides/self-hosting)
- [Coolify Documentation](https://coolify.io/docs)
- [Supabase GitHub Issues](https://github.com/supabase/supabase/issues)
