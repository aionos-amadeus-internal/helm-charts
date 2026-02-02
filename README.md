# Helm Charts Repository

Generic, modular Helm charts for Node.js, React (nginx static), and FastAPI applications. Use with K3s (e.g. Rancher) and GitHub Actions.

## Layout

```
chart/
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

## Override pattern (project-specific values)

Application repos keep a small values file (e.g. `deploy/values.yaml` or `helm-values.yaml`) that overrides:

- `nameOverride` / `fullnameOverride` (release name)
- `image.repository`, `image.tag`, `image.pullPolicy`
- `env` (environment variables)

**Merge order in Helm:**

1. Base: `chart/<stack>/values.yaml`
2. Env: `chart/<stack>/values-<env>.yaml`
3. App: your project's values file (last wins)

Example:

```bash
helm upgrade --install my-app ./chart/node \
  -f chart/node/values.yaml \
  -f chart/node/values-develop.yaml \
  -f ./deploy/values.yaml \
  -n my-namespace --create-namespace
```

## GitHub Actions usage

1. **Secret**: Add the chart repo URL as a repo secret (e.g. `HELM_CHARTS_REPO_URL`). Use HTTPS with a PAT or SSH.

2. **Checkout chart repo**: In the deploy job, checkout this repo (e.g. with `path: helm-charts` so chart path is `helm-charts/chart/<stack>`).

   Example (second checkout):

   ```yaml
   - name: Checkout charts
     uses: actions/checkout@v4
     with:
       repository: your-org/helm-charts
       token: ${{ secrets.HELM_CHARTS_REPO_TOKEN }}
       path: helm-charts
   ```

3. **Select chart and env**: Set `CHART_PATH=helm-charts/chart/node` (or `react` / `fastapi`) and `VALUES_ENV=develop` (or `release`, `pre-release`, `main`) from branch or inputs.

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
