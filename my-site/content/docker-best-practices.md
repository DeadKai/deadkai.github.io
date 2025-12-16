+++
title = "Docker Best Practices for Production"
date = 2024-12-01T14:30:00Z
+++

Docker has revolutionized how we build, ship, and run applications. However, creating production-ready Docker images requires following best practices to ensure security, efficiency, and reliability.

## Use Multi-Stage Builds

Multi-stage builds allow you to use multiple FROM statements in your Dockerfile, keeping your final image small by excluding build dependencies:

```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

This pattern reduces the final image size by 60-80% in many cases.

## Optimize Layer Caching

Docker builds images in layers, and each instruction creates a new layer. Order your Dockerfile to maximize cache hits:

```dockerfile
# Good: Dependencies change less frequently than source code
COPY package*.json ./
RUN npm ci
COPY . .

# Bad: Copying everything first invalidates cache on any file change
COPY . .
RUN npm ci
```

## Use Specific Base Image Tags

Never use the `latest` tag in production:

```dockerfile
# Bad
FROM node:latest

# Good
FROM node:18.17.1-alpine
```

Specific tags ensure reproducible builds and prevent unexpected breaking changes.

## Run as Non-Root User

Running containers as root is a security risk. Create and use a non-privileged user:

```dockerfile
FROM node:18-alpine

# Create app user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Set ownership
WORKDIR /app
COPY --chown=nodejs:nodejs . .

# Switch to non-root user
USER nodejs

CMD ["node", "index.js"]
```

## Minimize Image Size

Smaller images mean faster deployments and reduced attack surface:

1. **Use Alpine-based images** when possible (node:alpine, python:alpine)
2. **Remove unnecessary files** in the same layer they're created
3. **Use .dockerignore** to exclude files from the build context

```dockerignore
node_modules
npm-debug.log
.git
.env
*.md
tests
```

## Handle Secrets Securely

Never hardcode secrets in Dockerfiles or images:

```dockerfile
# Bad
ENV API_KEY=secret123

# Good: Use build-time secrets
RUN --mount=type=secret,id=api_key \
    API_KEY=$(cat /run/secrets/api_key) npm run configure
```

In production, use environment variables, Docker secrets, or dedicated secret management tools like Vault.

## Implement Health Checks

Health checks help orchestration platforms determine if your container is functioning correctly:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js || exit 1
```

## Use ENTRYPOINT and CMD Correctly

ENTRYPOINT defines the executable, while CMD provides default arguments:

```dockerfile
ENTRYPOINT ["node"]
CMD ["index.js"]
```

This allows users to override arguments: `docker run myimage server.js`

## Scan for Vulnerabilities

Regularly scan your images for security vulnerabilities:

```bash
docker scan myimage:latest
```

Tools like Trivy, Snyk, and Clair can integrate into CI/CD pipelines to catch vulnerabilities before deployment.

## Log to STDOUT/STDERR

Containers should log to standard output and error streams, not files:

```javascript
// Good
console.log('Server started');
console.error('Error occurred');

// Bad (in containers)
fs.appendFileSync('/var/log/app.log', 'Server started');
```

This integrates with Docker's logging drivers and centralized logging solutions.

## Conclusion

Following these best practices will result in Docker images that are secure, efficient, and production-ready. The key principles are:

- Keep images small
- Optimize for caching
- Follow security best practices
- Make containers observable
- Ensure reproducibility

By investing time in proper Dockerization, you'll save countless hours in debugging, security incidents, and deployment issues.
