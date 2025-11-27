# Kamal Boilerplate

A boilerplate project for deploying applications using [Kamal](https://kamal-deploy.org/) (formerly MRSK), a deployment tool from 37signals that allows you to deploy web applications to any server using Docker containers.

## Objective

This boilerplate provides a ready-to-use foundation for deploying containerized applications with Kamal. It includes:

- **Pre-configured deployment setup** with MySQL and Redis accessories
- **Environment variable management** through a wrapper script
- **Zero-downtime deployment** configuration

The goal is to eliminate the initial setup complexity and provide a tested starting point for deploying applications to your own servers, whether you're deploying a simple web app or a complex multi-service application.

## What is Kamal?

Kamal is a deployment tool that simplifies deploying containerized applications to your own servers. It handles:
- Building and pushing Docker images
- Managing container lifecycles
- Zero-downtime deployments
- SSL/TLS certificate management (via Let's Encrypt)
- Accessory services (databases, Redis, etc.)
- Environment variable management

## Project Structure

```
.
├── bin/
│   └── kamal                 # Kamal wrapper script (loads .env.kamal)
├── config/
│   ├── deploy.yml            # Main Kamal deployment configuration
├── src/
│   ├── Dockerfile            # Development Dockerfile
│   └── Dockerfile.prd        # Production Dockerfile
└── .kamal/
    └── secrets               # Environment secrets configuration
```

## The Kamal Wrapper Script

The `bin/kamal` file is a bash wrapper script that simplifies working with Kamal by automatically loading environment variables from `.env.kamal` before executing Kamal commands.

**How it works:**

```bash
#!/bin/bash

# Load environment variables from .env.kamal
if [ -f .env.kamal ]; then
  set -a
  source .env.kamal
  set +a
else
  echo "Warning: .env.kamal file not found"
fi

# Execute kamal with all passed arguments
kamal "$@"
```

**Benefits:**

- **No manual environment setup** - Variables are automatically injected
- **Consistent configuration** - All team members use the same environment file
- **Simple execution** - Just run `./bin/kamal deploy` instead of `export VAR=value && kamal deploy`
- **Version control friendly** - Add `.env.kamal` to `.gitignore` to keep secrets out of git

**Usage:**

```bash
# Instead of:
export APP_NAME=my-app
export SERVER_IP=192.168.0.1
kamal deploy

# Simply run:
./bin/kamal deploy
```

All environment variables are loaded automatically from `.env.kamal`, making deployment commands cleaner and more reproducible.

## Prerequisites

- Ruby (for running Kamal)
- Docker
- SSH access to your deployment server
- A container registry (Docker Hub, GitHub Container Registry, or a private registry)

## Setup Instructions

### 1. Install Kamal

If you don't have Kamal installed globally, you can use the bundled version:

```bash
gem install kamal
```

Or use the local bin version:

```bash
./bin/kamal version
```

### 2. Configure Environment Variables

Create a `.env.kamal` file in the root directory with the following variables:

```bash
# Application Configuration
export APP_NAME="my-app"
export IMAGE_NAME="my-app"
export APP_HOST="yourdomain.com"

# Server Configuration
export SERVER_IP="192.168.0.1"
export SSH_USER="root"
export SSH_PORT="22"
export SSH_KEY_PATH="~/.ssh/id_rsa"

# Registry Configuration
export REGISTRY_SERVER="localhost:5555"  # or docker.io, ghcr.io, etc.
export REGISTRY_USERNAME="your-username"
export KAMAL_REGISTRY_PASSWORD="your-registry-password"

# Database Configuration
export MYSQL_ROOT_PASSWORD="your-secure-password"
```

**Important:** Add `.env.kamal` to your `.gitignore` file to prevent committing sensitive credentials:

```bash
echo ".env.kamal" >> .gitignore
```

The wrapper script will automatically load these variables when you run any `./bin/kamal` command.

### 3. Configure Secrets

Edit `.kamal/secrets` to define how secrets are loaded. The file supports three methods:

**Option 1: From Environment Variables**
```bash
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD
MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD
```

**Option 2: From Files**
```bash
RAILS_MASTER_KEY=$(cat config/master.key)
```

**Option 3: From Password Managers**
```bash
SECRETS=$(kamal secrets fetch --adapter 1password --account my-account --from MyVault/MyItem KAMAL_REGISTRY_PASSWORD)
KAMAL_REGISTRY_PASSWORD=$(kamal secrets extract KAMAL_REGISTRY_PASSWORD $SECRETS)
```

### 4. Customize Your Application

Replace the sample Dockerfile in `src/Dockerfile.prd` with your actual application Dockerfile.

Current example (Alpine Linux with Hello World):
```dockerfile
FROM alpine:latest
RUN echo "Hello Kamal Boilerplate Hello World"
CMD ["sh", "-c", "echo 'Hello Kamal Boilerplate Hello World' && tail -f /dev/null"]
```

## How to Use

### Initial Setup

First, set up Kamal on your server:

```bash
./bin/kamal setup
```

This will:
- Install Docker on the server if not present
- Set up the Kamal proxy (Traefik)
- Create necessary directories
- Start accessory services (MySQL, Redis)

### Deploy Your Application

Deploy your application:

```bash
./bin/kamal deploy
```

This will:
1. Build your Docker image
2. Push it to your registry
3. Pull the image on your server
4. Start the container with zero downtime
5. Update the proxy configuration

### Common Commands

```bash
# Check application status
./bin/kamal app status

# View application logs
./bin/kamal app logs

# Access application shell
./bin/kamal app exec --interactive bash

# Restart the application
./bin/kamal app restart

# Stop the application
./bin/kamal app stop

# View all containers
./bin/kamal app containers

# Deploy a specific version
./bin/kamal deploy --version v1.2.3
```

### Managing Accessories

```bash
# Check accessory status
./bin/kamal accessory status

# View database logs
./bin/kamal accessory logs db

# Access MySQL shell
./bin/kamal accessory exec db mysql -u root -p

# Restart Redis
./bin/kamal accessory restart redis
```

### Rollback

If something goes wrong:

```bash
./bin/kamal rollback [VERSION]
```

## How It Works

### Deployment Process

1. **Build Phase**
   - Kamal builds your Docker image using the Dockerfile specified in `config/deploy.yml`
   - The build happens remotely on your server via SSH
   - Build arguments and secrets are injected as needed

2. **Push Phase**
   - The built image is tagged with a unique identifier
   - Image is pushed to your configured registry

3. **Deploy Phase**
   - Kamal pulls the new image to your server
   - Starts new containers with the updated image
   - Performs health checks
   - Once healthy, traffic is redirected to new containers
   - Old containers are stopped and removed

4. **Proxy Management**
   - Traefik proxy routes traffic to your containers
   - Handles SSL/TLS certificates (if enabled)
   - Enables multiple apps on a single server

### Accessory Services

The boilerplate includes two accessory services:

**MySQL 8.0**
- Persistent data storage in `data:/var/lib/mysql`
- Custom configuration via `config/mysql/production.cnf`
- Initialization script from `db/production.sql`

**Redis (Valkey 8)**
- Persistent data storage in `data:/data`
- Port 6379 exposed for connections

### Volumes

Persistent storage is configured for:
- Application storage: `/var/www/storage/app/public`
- MySQL data: `/var/lib/mysql`
- Redis data: `/data`

### Asset Management

The boilerplate bridges fingerprinted assets between deployments to avoid 404 errors during rolling updates. Assets in `/var/www/public/assets` are preserved across versions.

## Configuration Options

### SSL/TLS

To enable SSL with Let's Encrypt, update `config/deploy.yml`:

```yaml
proxy:
  ssl: true
  host: yourdomain.com
```

**Note:** If using Cloudflare, set SSL/TLS encryption mode to "Full".

### Multiple Servers

For production environments with multiple servers, configure server groups:

```yaml
servers:
  web:
    - 192.168.0.1
    - 192.168.0.2
  worker:
    hosts:
      - 192.168.0.3
    cmd: bin/worker
```

### Rolling Deploys

Configure gradual rollouts:

```yaml
boot:
  limit: 10  # or "25%" for percentage
  wait: 2    # seconds between batches
```

## Troubleshooting

### Check Kamal Configuration

```bash
./bin/kamal config
```

### View Server Details

```bash
./bin/kamal details
```

### SSH into Server

```bash
./bin/kamal app exec --interactive bash
```

### View Full Logs

```bash
./bin/kamal app logs --tail 100
```

### Clear and Redeploy

```bash
./bin/kamal remove
./bin/kamal setup
./bin/kamal deploy
```

## Security Notes

- **Never commit secrets:** The `.kamal/secrets` file should reference environment variables, not contain actual credentials
- **Use SSH keys:** Configure key-based authentication instead of passwords
- **Restrict registry access:** Use private registries with proper authentication
- **Keep Docker updated:** Regularly update Docker on your servers

## Resources

- [Official Kamal Documentation](https://kamal-deploy.org/)
- [Kamal GitHub Repository](https://github.com/basecamp/kamal)
- [Docker Documentation](https://docs.docker.com/)
- [Traefik Proxy Documentation](https://doc.traefik.io/traefik/)

## License

This boilerplate is provided as-is for use in your own projects.
