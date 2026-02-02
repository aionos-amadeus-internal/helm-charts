# Ingress Configuration Examples

This document shows how to configure Ingress for public access to your applications.

## Frontend (React) - Example

In your app's `deploy/values-develop.yaml`:

```yaml
nameOverride: my-frontend
ingress:
  enabled: true
  className: "traefik"
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
```

## API (Node.js) - Example

In your app's `deploy/values-develop.yaml`:

```yaml
nameOverride: my-api
ingress:
  enabled: true
  className: "traefik"
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  # Optional: Add API-specific annotations
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-cors@kubernetescrd
```

## FastAPI - Example

In your app's `deploy/values-develop.yaml`:

```yaml
nameOverride: my-fastapi
ingress:
  enabled: true
  className: "traefik"
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  # Optional: Add CORS or other middleware annotations
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-cors@kubernetescrd
```

## TLS/HTTPS Configuration

To enable HTTPS, add TLS configuration:

```yaml
ingress:
  enabled: true
  className: "traefik"
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls-secret
      hosts:
        - app.example.com
```

**Note:** You need to create the TLS secret manually or use cert-manager for automatic certificate management.

## Path-based Routing

For APIs under a subpath:

```yaml
ingress:
  enabled: true
  className: "traefik"
  hosts:
    - host: example.com
      paths:
        - path: /api
          pathType: Prefix
```

## Multiple Hosts

To serve the same app on multiple domains:

```yaml
ingress:
  enabled: true
  className: "traefik"
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
    - host: www.example.com
      paths:
        - path: /
          pathType: Prefix
```

## Traefik-specific Annotations

Common Traefik annotations you might want to use:

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    # Enable CORS
    traefik.ingress.kubernetes.io/router.middlewares: default-cors@kubernetescrd
    # Add custom headers
    traefik.ingress.kubernetes.io/router.middlewares: default-headers@kubernetescrd
    # Rate limiting
    traefik.ingress.kubernetes.io/router.middlewares: default-ratelimit@kubernetescrd
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
```

## DNS Configuration

After deploying with Ingress enabled:

1. Get your k3s cluster's external IP or load balancer IP:
   ```bash
   kubectl get svc -n kube-system traefik
   ```

2. Point your domain's A record to that IP:
   - `agent-studio.aionos.co` → `<cluster-ip>`

3. Traefik will automatically route traffic to your service based on the Host header.
