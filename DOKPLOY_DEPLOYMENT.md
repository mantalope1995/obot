# Obot Deployment Guide for Dokploy

This guide walks you through deploying Obot using Dokploy, a self-hosted PaaS platform.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Pre-Deployment Setup](#pre-deployment-setup)
3. [Deploying to Dokploy](#deploying-to-dokploy)
4. [Post-Deployment Configuration](#post-deployment-configuration)
5. [Monitoring and Maintenance](#monitoring-and-maintenance)
6. [Troubleshooting](#troubleshooting)

## Prerequisites

### Required Software/Services

- **Dokploy** instance running on your server
- **Domain name** configured with DNS pointing to your Dokploy server
- **SSL/TLS certificate** (Dokploy can auto-generate with Let's Encrypt)
- **At least 4GB RAM** and **2 CPU cores** available for the deployment
- **20GB free disk space** for data storage

### Required API Keys

- **OpenAI API Key** (required for AI models) - Get from [platform.openai.com](https://platform.openai.com/)
- Optional: **Anthropic API Key** for Claude models - Get from [console.anthropic.com](https://console.anthropic.com/)

## Pre-Deployment Setup

### 1. Prepare Your Environment

#### Generate Required Secrets

```bash
# Generate a strong PostgreSQL password
openssl rand -base64 32

# Generate an encryption key for Obot
openssl rand -base64 32

# Generate a bootstrap token (optional, Obot can generate one)
openssl rand -base64 32
```

**Save these values securely** - you'll need them for the `.env` file.

### 2. Configure Environment Variables

Copy the example environment file:

```bash
cp .env.example .env
```

Edit the `.env` file and update the following **critical** variables:

```bash
# Database Configuration
POSTGRES_PASSWORD=your_generated_postgres_password
OBOT_SERVER_DSN=postgres://obot:your_generated_postgres_password@postgres:5432/obot

# Application Configuration
OBOT_SERVER_HOSTNAME=https://your-actual-domain.com
OBOT_SERVER_ENABLE_AUTHENTICATION=true
OBOT_SERVER_AUTH_OWNER_EMAILS=your-email@domain.com

# AI Model Configuration
OPENAI_API_KEY=your-actual-openai-api-key

# Encryption Configuration
OBOT_SERVER_ENCRYPTION_KEY=your_generated_encryption_key
```

**Important Security Notes:**
- Use strong, unique passwords
- Never commit the `.env` file to version control
- Keep API keys secure and rotate them regularly
- Enable authentication for production deployments

## Deploying to Dokploy

### Step 1: Create a New Application

1. Log in to your Dokploy dashboard
2. Click **"Create Application"** or **"New Project"**
3. Select **"Docker Compose"** as the application type
4. Name your application (e.g., "obot")

### Step 2: Configure the Docker Compose

1. In the application configuration, you'll see a **"Docker Compose"** section
2. Copy the contents of `docker-compose.yml` into the editor
3. **Important**: Review and adjust the following based on your server capacity:

   ```yaml
   deploy:
     resources:
       limits:
         cpus: '2.0'      # Adjust based on available CPU
         memory: 4G       # Adjust based on available RAM
       reservations:
         cpus: '0.5'
         memory: 1G
   ```

4. **Update port mapping** if needed:
   ```yaml
   ports:
     - "8080:8080"  # External port:Internal port
   ```

### Step 3: Add Environment Variables

In Dokploy's environment variables section:

1. **Method A**: Upload the `.env` file
   - Look for an option to upload an `.env` file
   - Upload your configured `.env` file

2. **Method B**: Add variables manually
   - Copy each variable from your `.env` file
   - Add them to Dokploy's environment variables section
   - Ensure sensitive values are marked as secrets (Dokploy typically has a "secret" checkbox)

### Step 4: Configure Domain and SSL

1. Navigate to the **"Domains"** or **"Networking"** section
2. Add your domain (e.g., `obot.yourdomain.com`)
3. Enable **HTTPS/SSL**
4. Configure **Let's Encrypt** for automatic certificate generation
5. Set up a reverse proxy to forward traffic to port 8080

**Example Nginx Configuration** (if using Dokploy's built-in Nginx):

```nginx
server {
    listen 80;
    server_name obot.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name obot.yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Step 5: Deploy the Application

1. Click **"Deploy"** or **"Start"** in Dokploy
2. Monitor the deployment logs:
   - Watch for PostgreSQL initialization
   - Wait for Obot to connect to the database
   - Verify the health checks pass
3. The application should be accessible at your configured domain

## Post-Deployment Configuration

### 1. Initial Setup

1. Open your browser and navigate to `https://your-domain.com`
2. If you set `OBOT_BOOTSTRAP_TOKEN`, enter it
3. If not, check the logs to find the auto-generated token:

```bash
# View Obot logs in Dokploy
# Look for a line like: "Bootstrap token: your-token-here"
```

4. Create your admin account using the bootstrap token
5. Configure your model providers (OpenAI, Anthropic, etc.)

### 2. Configure Model Providers

1. Navigate to **Settings** → **Model Providers**
2. Add OpenAI:
   - Provider: OpenAI
   - API Key: Your OpenAI API key
   - Models: Select desired models (e.g., gpt-4, gpt-3.5-turbo)
3. (Optional) Add Anthropic for Claude models

### 3. Configure MCP Catalog

1. Navigate to **MCP Registry** → **Catalog**
2. The default catalog will be loaded from GitHub
3. You can:
   - Browse available MCP servers
   - Add custom MCP servers
   - Configure access control rules

### 4. Set Up Users and Permissions

1. Navigate to **Settings** → **Users**
2. Create additional users if needed
3. Assign appropriate roles:
   - **Owner**: Full administrative access
   - **Admin**: Can manage most settings
   - **User**: Can use features but limited management

### 5. Configure Security

1. Enable **audit logging** if required by your organization:
   ```bash
   OBOT_SERVER_AUDIT_LOGS_MODE=disk
   ```

2. Set up **data retention policies**:
   ```bash
   OBOT_SERVER_RETENTION_POLICY_HOURS=2160  # 90 days
   ```

3. Consider enabling **S3 audit log storage** for long-term compliance

## Monitoring and Maintenance

### Health Checks

The docker-compose configuration includes health checks:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

Monitor these in Dokploy's dashboard.

### Viewing Logs

Access logs through Dokploy:
1. Navigate to your application
2. Click on the **Logs** tab
3. Select the service (`obot` or `postgres`)
4. Monitor for errors or warnings

### Backup Strategy

#### Database Backups

Set up automated PostgreSQL backups:

```bash
# Create a backup script
cat > /opt/obot/backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/opt/obot/backups"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

# Backup PostgreSQL
docker exec obot-postgres pg_dump -U obot obot > $BACKUP_DIR/backup_$DATE.sql

# Keep last 7 days of backups
find $BACKUP_DIR -name "backup_*.sql" -mtime +7 -delete
EOF

chmod +x /opt/obot/backup.sh
```

Add to crontab for daily backups:
```bash
# Run daily at 2 AM
0 2 * * * /opt/obot/backup.sh
```

#### Data Volume Backups

Backup the data directories:
```bash
tar -czf /opt/obot/data_backup_$(date +%Y%m%d).tar.gz /opt/obot/data
tar -czf /opt/obot/postgres_backup_$(date +%Y%m%d).tar.gz /opt/obot/postgres
```

### Updating Obot

When a new version is available:

1. Update the image tag in `docker-compose.yml`:
   ```yaml
   image: ghcr.io/obot-platform/obot:latest  # Or specific version tag
   ```

2. Redeploy in Dokploy:
   - Click **"Redeploy"** or **"Update"**
   - Monitor the deployment logs
   - Verify the application is running correctly

### Resource Scaling

If you need more resources:

1. Update resource limits in `docker-compose.yml`:
   ```yaml
   deploy:
     resources:
       limits:
         cpus: '4.0'      # Increase CPU
         memory: 8G       # Increase RAM
   ```

2. Redeploy the application

## Troubleshooting

### Application Won't Start

**Symptoms**: Container exits immediately or fails health checks

**Solutions**:

1. Check the logs:
   ```bash
   # In Dokploy, view the application logs
   # Look for error messages about database connection or configuration
   ```

2. Verify environment variables:
   - Ensure `OBOT_SERVER_DSN` is correct
   - Check that PostgreSQL is running and healthy
   - Verify API keys are properly set

3. Check database connectivity:
   ```bash
   # Test PostgreSQL connection
   docker exec obot-postgres psql -U obot -d obot -c "SELECT version();"
   ```

### Database Connection Issues

**Symptoms**: Obot cannot connect to PostgreSQL

**Solutions**:

1. Verify PostgreSQL is healthy:
   ```bash
   docker exec obot-postgres pg_isready -U obot
   ```

2. Check network connectivity:
   ```bash
   docker exec obot ping postgres
   ```

3. Verify DSN format:
   ```
   postgres://username:password@hostname:port/database
   ```

### Permission Errors

**Symptoms**: Cannot write to data directories

**Solutions**:

1. Check directory permissions:
   ```bash
   ls -la /opt/obot/data
   ls -la /opt/obot/postgres
   ```

2. Fix permissions:
   ```bash
   sudo chown -R 1000:1000 /opt/obot/data
   sudo chown -R 70:70 /opt/obot/postgres
   ```

### High Memory Usage

**Symptoms**: Server runs out of RAM

**Solutions**:

1. Reduce memory limits in `docker-compose.yml`
2. Decrease worker counts:
   ```bash
   NAH_THREADINESS=5000
   OBOT_SERVER_KNOWLEDGE_FILE_WORKERS=3
   KINM_DB_CONNECTIONS=3
   ```
3. Monitor and limit MCP server instances

### Cannot Access from Browser

**Symptoms**: Connection refused or timeout

**Solutions**:

1. Verify the application is running:
   ```bash
   docker ps | grep obot
   ```

2. Check port mapping:
   ```bash
   netstat -tlnp | grep 8080
   ```

3. Verify reverse proxy configuration
4. Check firewall rules:
   ```bash
   sudo ufw status
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   ```

### MCP Servers Not Starting

**Symptoms**: MCP servers fail to deploy

**Solutions**:

1. Verify Docker socket is mounted:
   ```bash
   docker exec obot ls -la /var/run/docker.sock
   ```

2. Check MCP base image:
   ```bash
   OBOT_SERVER_MCPBASE_IMAGE=ghcr.io/obot-platform/mcp-images/phat:main
   ```

3. Verify sufficient resources:
   - Check CPU and memory availability
   - Increase resource limits if needed

## Production Checklist

Before going to production, ensure:

- [ ] Strong passwords are set for all credentials
- [ ] SSL/TLS certificates are properly configured
- [ ] Authentication is enabled
- [ ] Encryption is configured for sensitive data
- [ ] Backup strategy is in place and tested
- [ ] Monitoring and alerting are configured
- [ ] Resource limits are appropriate for your workload
- [ ] Audit logging is enabled (if required)
- [ ] Data retention policy is set
- [ ] Domain and DNS are properly configured
- [ ] Firewall rules are configured
- [ ] Health checks are passing
- [ ] Documentation is updated with deployment details

## Additional Resources

- [Obot Documentation](https://docs.obot.ai)
- [Obot GitHub Repository](https://github.com/obot-platform/obot)
- [MCP Protocol Specification](https://modelcontextprotocol.io)
- [Dokploy Documentation](https://dokploy.com/docs)

## Support

- **Documentation**: [docs.obot.ai](https://docs.obot.ai)
- **Discord Community**: [Join the Discord](https://discord.com/invite/9sSf4UyAMC)
- **GitHub Issues**: [Report bugs or request features](https://github.com/obot-platform/obot/issues)
