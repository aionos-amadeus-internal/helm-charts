# Helm Charts Repository

Generic, modular Helm charts for Node.js, React (nginx static), and FastAPI applications. Use with K3s (e.g. Rancher) and GitHub Actions.

## Table of Contents

- [Layout](#layout)
- [Environments](#environments)
- [Override Pattern](#override-pattern-project-specific-values)
- [GitHub Actions Usage](#github-actions-usage)
- [Example App Values File](#example-app-values-file)
- [Resources](#resources-defaults)
- [Ingress Configuration](#ingress-configuration)
- [Let's Encrypt HTTPS Setup](#lets-encrypt-https-setup)
- [cert-manager TLS Pattern](#cert-manager-tls-pattern-locked-configuration)
- [Troubleshooting](#troubleshooting)

---

## Layout

```
node/      # Node.js apps (default port 3000)
react/     # React static apps served by nginx (port 80)
fastapi/   # FastAPI apps (port 8000)
```

Each chart is standalone: `Chart.yaml`, `values.yaml`, env-specific value files, and `templates/`.

## Environments

Value files per environment (same minimal resources for all for now):

- `values-develop.yaml`
- `values-release.yaml`
- `values-pre-release.yaml`
- `values-main.yaml`

## Override Pattern (project-specific values)

Charts are **generic**: no project-specific hostnames, domains, or TLS. Application repos own all of that via **per-env values** (e.g. `deploy/values-develop.yaml`, `deploy/values-main.yaml`).

Application values override:

- `nameOverride` / `fullnameOverride` (release name)
- `image.repository`, `image.tag`, `image.pullPolicy`
- **`ingress.enabled`, `ingress.hosts`, `ingress.tls`, `ingress.annotations`** (host and TLS **must** be set here, not in the chart)
- `env` (environment variables)

**Merge order in Helm:**

1. Base: `<stack>/values.yaml`
2. Env: `<stack>/values-<env>.yaml`
3. App: your project's values file (last wins)

Example:

```bash
helm upgrade --install my-app ./node \
  -f node/values.yaml \
  -f node/values-develop.yaml \
  -f ./deploy/values.yaml \
  -n my-namespace --create-namespace
```

## GitHub Actions usage

1. **Secret**: Add the chart repo URL as a repo secret (e.g. `HELM_CHARTS_REPO_URL`). Use HTTPS with a PAT or SSH.

2. **Checkout chart repo**: In the deploy job, checkout this repo (e.g. with `path: helm-charts` so chart path is `helm-charts/<stack>`).

   Example (second checkout):

   ```yaml
   - name: Checkout charts
     uses: actions/checkout@v4
     with:
       repository: your-org/helm-charts
       token: ${{ secrets.HELM_CHARTS_REPO_TOKEN }}
       path: helm-charts
   ```

3. **Select chart and env**: Set `CHART_PATH=helm-charts/node` (or `react` / `fastapi`) and `VALUES_ENV=develop` (or `release`, `pre-release`, `main`) from branch or inputs.

4. **Deploy**:

   ```bash
   helm upgrade --install $RELEASE_NAME $CHART_PATH \
     -f $CHART_PATH/values.yaml \
     -f $CHART_PATH/values-$VALUES_ENV.yaml \
     -f ./deploy/values.yaml \
     -n $NAMESPACE --create-namespace
   ```

Ensure the runner has `kubectl` context pointing at your K3s cluster (e.g. kubeconfig from secret or self-hosted runner with cluster access).

## Example app values file

In your application repo, e.g. `deploy/values.yaml`:

```yaml
nameOverride: my-app
image:
  repository: my-registry.io/my-app
  tag: latest
env:
  NODE_ENV: production
  API_URL: https://api.example.com
```

## Resources (defaults)

| Chart   | CPU (request / limit) | Memory (request / limit) | Notes                          |
|---------|------------------------|---------------------------|--------------------------------|
| **node**   | 100m / 500m            | 128Mi / 512Mi             | Typical Node.js API workload   |
| **react**  | 50m / 200m             | 64Mi / 256Mi              | Nginx static; minimal          |
| **fastapi**| 200m / 1000m           | 512Mi / 1Gi               | For Playwright/Chromium stacks |

Replicas: 1. Override in your app values or env-specific value files when you need to diverge per environment.

---

## Ingress Configuration

All charts support Ingress for public access. Enable it in your app-specific values.

### Frontend (React) - Example

In your app's `deploy/values-develop.yaml`:

```yaml
nameOverride: my-frontend
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: my-frontend-tls
      hosts:
        - app.example.com
```

### API (Node.js) - Example

In your app's `deploy/values-develop.yaml`:

```yaml
nameOverride: my-api
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    # Optional: Add API-specific middleware annotations
    # traefik.ingress.kubernetes.io/router.middlewares: default-cors@kubernetescrd
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: my-api-tls
      hosts:
        - api.example.com
```

### FastAPI - Example

In your app's `deploy/values-develop.yaml`:

```yaml
nameOverride: my-fastapi
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    # Optional: Add CORS or other middleware annotations
    # traefik.ingress.kubernetes.io/router.middlewares: default-cors@kubernetescrd
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: my-fastapi-tls
      hosts:
        - api.example.com
```

### Path-based Routing

For APIs under a subpath:

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: example.com
      paths:
        - path: /api
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - example.com
```

### Multiple Hosts

To serve the same app on multiple domains:

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
    - host: www.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-tls
      hosts:
        - app.example.com
        - www.example.com
```

### Traefik-specific Annotations

Common Traefik annotations you might want to use (alongside cert-manager):

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    # Required: cert-manager for HTTPS
    cert-manager.io/cluster-issuer: letsencrypt-prod
    # Optional: Traefik middleware
    # traefik.ingress.kubernetes.io/router.middlewares: default-cors@kubernetescrd
    # traefik.ingress.kubernetes.io/router.middlewares: default-headers@kubernetescrd
    # traefik.ingress.kubernetes.io/router.middlewares: default-ratelimit@kubernetescrd
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.example.com
```

**Note:** Do NOT use `traefik.ingress.kubernetes.io/router.tls.certresolver` - that's for Traefik's built-in ACME, not cert-manager.

### DNS Configuration

After deploying with Ingress enabled:

1. Get your k3s cluster's external IP or load balancer IP:
   ```bash
   kubectl get svc -n kube-system traefik
   ```

2. Point your domain's A record to that IP:
   - `agent-studio.aionos.co` → `<cluster-ip>`

3. Traefik will automatically route traffic to your service based on the Host header.

---

## Let's Encrypt HTTPS Setup

This guide explains how to enable HTTPS with Let's Encrypt certificates using **cert-manager**.

### Setup Overview

**This cluster uses cert-manager with ClusterIssuer for Let's Encrypt.**

- ✅ cert-manager is installed cluster-wide
- ✅ ClusterIssuer (`letsencrypt-prod`) exists
- ✅ TLS is enabled **per Ingress** (not cluster-wide)
- ✅ Each app must declare TLS in its values file

### Configuration Pattern

For **every application**, use this pattern:

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: your-domain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: your-app-tls
      hosts:
        - your-domain.com
```

### Required Components

1. **cert-manager annotation**: `cert-manager.io/cluster-issuer: letsencrypt-prod`
2. **TLS section**: Must include `secretName` and `hosts`
3. **Host match**: TLS `hosts` must match ingress `hosts`

### How It Works

When you deploy with this configuration:

1. **Helm creates Ingress** with TLS section
2. **cert-manager detects** the annotation and TLS section
3. **cert-manager creates Certificate** resource automatically
4. **cert-manager solves ACME HTTP-01 challenge** via Traefik
5. **cert-manager creates TLS secret** (`your-app-tls`)
6. **Traefik detects TLS secret** and enables HTTPS
7. **cert-manager renews** certificates automatically (every 60 days)

**No manual steps needed!** cert-manager handles everything.

### Verify ClusterIssuer

Check your ClusterIssuer name:

```bash
kubectl get clusterissuer
```

Common names: `letsencrypt-prod`, `letsencrypt-staging`, `letsencrypt`

Update the annotation if your ClusterIssuer has a different name.

### Verification

After deployment, verify HTTPS:

1. **Check the Ingress:**
   ```bash
   kubectl get ingress -n agent-studio-frontend-dev
   kubectl describe ingress -n agent-studio-frontend-dev
   ```

2. **Check TLS secret:**
   ```bash
   kubectl get secret dev-agent-studio-tls -n agent-studio-frontend-dev
   ```

3. **Test HTTPS:**
   ```bash
   curl -I https://dev.agent-studio.aionos.co
   ```

4. **Check certificate:**
   ```bash
   openssl s_client -connect dev.agent-studio.aionos.co:443 -servername dev.agent-studio.aionos.co
   ```

### Important Notes

#### Do NOT Mix cert-manager and Traefik ACME

❌ **Never use both together:**
```yaml
annotations:
  cert-manager.io/cluster-issuer: letsencrypt-prod
  traefik.ingress.kubernetes.io/router.tls.certresolver: letsencrypt  # DON'T ADD THIS
```

✅ **Always use cert-manager only:**
```yaml
annotations:
  cert-manager.io/cluster-issuer: letsencrypt-prod
```

#### TLS is Per-Ingress, Not Cluster-Wide

- ClusterIssuer is cluster-wide (available to all namespaces)
- But **each Ingress must opt-in** to TLS
- Each app needs its own TLS section in values file
- Each app gets its own TLS secret

---

## cert-manager TLS Pattern (Locked Configuration)

### Cluster Setup

✅ **cert-manager** installed cluster-wide  
✅ **ClusterIssuer** (`letsencrypt-prod`) exists  
✅ **Traefik** as ingress controller  
✅ TLS enabled **per Ingress** (not cluster-wide)

### Required Pattern for Every App

**Every Helm chart values file MUST include:**

```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: your-domain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: your-app-tls
      hosts:
        - your-domain.com
```

### What Happens Automatically

1. Helm creates Ingress with TLS section
2. cert-manager detects annotation + TLS section
3. cert-manager creates Certificate resource
4. cert-manager solves ACME HTTP-01 challenge
5. cert-manager creates TLS secret
6. Traefik enables HTTPS automatically
7. Certificates auto-renew every 60 days

### Critical Rules

#### ✅ DO

- Always include `cert-manager.io/cluster-issuer` annotation
- Always include TLS section with `secretName` and `hosts`
- Match TLS `hosts` with ingress `hosts`
- Use unique `secretName` per app/environment

#### ❌ DON'T

- **Never** use Traefik certresolver annotations (`traefik.ingress.kubernetes.io/router.tls.certresolver`)
- **Never** mix cert-manager and Traefik ACME
- **Never** create TLS secrets manually (cert-manager does it)
- **Never** assume TLS is cluster-wide (it's per-Ingress)

### Examples by Chart Type

#### React Frontend
```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: app-frontend-tls
      hosts:
        - app.example.com
```

#### Node.js API
```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-node-tls
      hosts:
        - api.example.com
```

#### FastAPI
```yaml
ingress:
  enabled: true
  className: "traefik"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-fastapi-tls
      hosts:
        - api.example.com
```

### Verification Commands

```bash
# Check ClusterIssuer
kubectl get clusterissuer

# Check Certificate resources
kubectl get certificates -A

# Check TLS secrets
kubectl get secrets -A | grep tls

# Check Ingress
kubectl get ingress -A

# Test HTTPS
curl -I https://your-domain.com
```

---

## Troubleshooting

### Certificate not issued

```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# Check Certificate resources
kubectl get certificates -A

# Check Certificate details
kubectl describe certificate <secret-name> -n <namespace>

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager
```

**Common issues:**
- DNS not pointing to cluster → Fix DNS A record
- Let's Encrypt rate limit → Wait or use staging issuer
- Wrong ClusterIssuer name → Check `kubectl get clusterissuer`
- Missing TLS section → Add TLS section to values file

### HTTPS not working

- Verify TLS secret exists: `kubectl get secret <secret-name> -n <namespace>`
- Check Ingress has TLS section: `kubectl get ingress -n <namespace> -o yaml`
- Verify Traefik is routing: `kubectl logs -n kube-system -l app.kubernetes.io/name=traefik`
- Test DNS: `dig your-domain.com`

### Certificate not created

- Verify annotation: `cert-manager.io/cluster-issuer: letsencrypt-prod`
- Check TLS section exists with matching hosts
- Verify DNS points to cluster
- Check cert-manager logs: `kubectl logs -n cert-manager -l app=cert-manager`

---

**This pattern is locked. Use it for all new applications.**
