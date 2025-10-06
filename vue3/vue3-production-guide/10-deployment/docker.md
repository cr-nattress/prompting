# Docker Containerization

> **File Purpose**: Containerize Vue 3 app with multi-stage Docker build
> **Prerequisites**: `vite-build.md`
> **Agent Use Case**: Reference when creating production Docker images
> **Related Files**: `ci-cd.md`

## In One Sentence

Use a multi-stage Dockerfile to build the Vue app in a Node container and serve it from a lightweight Nginx container.

## Multi-Stage Dockerfile

```dockerfile
# Stage 1: Build the application
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY . .

# Build for production
RUN npm run build

# Stage 2: Serve with Nginx
FROM nginx:alpine

# Copy custom nginx config
COPY nginx.conf /etc/nginx/nginx.conf

# Copy built assets from builder stage
COPY --from=builder /app/dist /usr/share/nginx/html

# Expose port 80
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

## Nginx Configuration

Create `nginx.conf` in project root:

```nginx
events {
  worker_connections 1024;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  # Gzip compression
  gzip on;
  gzip_vary on;
  gzip_proxied any;
  gzip_comp_level 6;
  gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss;

  server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # Security headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;

    # SPA fallback: serve index.html for all routes
    location / {
      try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location /assets/ {
      expires 1y;
      add_header Cache-Control "public, immutable";
    }

    # Don't cache index.html
    location = /index.html {
      add_header Cache-Control "no-cache";
    }
  }
}
```

## .dockerignore

```
node_modules
dist
.git
.github
.vscode
*.md
.env.local
.env.*.local
npm-debug.log*
yarn-debug.log*
yarn-error.log*
```

## Build and Run

```bash
# Build image
docker build -t my-vue-app:latest .

# Run container
docker run -p 8080:80 my-vue-app:latest

# Access at http://localhost:8080
```

## Docker Compose (for local development with backend)

```yaml
# docker-compose.yml
version: '3.8'

services:
  frontend:
    build: .
    ports:
      - "8080:80"
    depends_on:
      - backend

  backend:
    image: my-backend-api:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
```

Run: `docker-compose up`

## Environment Variables in Docker

```dockerfile
# Build with build args
FROM node:20-alpine AS builder

ARG VITE_API_BASE_URL
ENV VITE_API_BASE_URL=$VITE_API_BASE_URL

# ... rest of Dockerfile
```

Build with args:

```bash
docker build --build-arg VITE_API_BASE_URL=https://api.example.com -t my-vue-app:latest .
```

---

## Navigation

- **Previous**: `10-deployment/vite-build.md`
- **Next**: `10-deployment/ci-cd.md`
- **Up**: `00-overview.md`
