# taskcluster-image-sync

Syncs Taskcluster Docker images from Docker Hub to Google Artifact Registry (GAR) for MozCloud deployments.

This enables ArgoCD to monitor GAR for new image tags and trigger automatic deployments to lower environments, as well as allowing version selection from the ArgoCD UI for production deployments.

See [taskcluster/taskcluster#7925](https://github.com/taskcluster/taskcluster/issues/7925) for background.

## How It Works

A GitHub Actions workflow runs every 30 minutes and:

1. Queries Docker Hub for recent release tags of each source image (supports both `v`-prefixed and unprefixed tags)
2. Queries GAR for existing tags (stateless diff — GAR is the source of truth)
3. Pulls any missing tags from Docker Hub, retags them for GAR, and pushes them

### Images Synced

| Docker Hub | GAR |
|------------|-----|
| `taskcluster/taskcluster` | `<GAR_REGISTRY>/taskcluster` |
| `taskcluster/websocktunnel` | `<GAR_REGISTRY>/websocktunnel` |

## Manual Sync

To sync a specific version immediately (e.g., for an urgent deployment):

1. Go to **Actions** > **Sync Taskcluster Images to GAR**
2. Click **Run workflow**
3. Enter the version tag (e.g., `v98.0.1` or `98.0.1`) or leave blank to run the normal diff sync
4. Click **Run workflow**

Manual runs with a version input will force-sync that tag even if it already exists in GAR.

## Adding a New GAR Target

1. Add WIF secrets to the repo:
   - `GCP_WORKLOAD_IDENTITY_PROVIDER_<ENV>` — Workload Identity Provider resource name
   - `GCP_SERVICE_ACCOUNT_EMAIL_<ENV>` — Service account email with GAR write access
2. Add a new entry to the `matrix.include` list in `.github/workflows/sync-images.yml`

## Adding a New Image

Add the Docker Hub image path (e.g., `taskcluster/new-image`) to the `SOURCE_IMAGES` env var in `.github/workflows/sync-images.yml`.

## Required Secrets

| Secret | Description | Required |
|--------|-------------|----------|
| `GCP_WORKLOAD_IDENTITY_PROVIDER_PROD` | WIF provider for prod GAR | Yes |
| `GCP_SERVICE_ACCOUNT_EMAIL_PROD` | Service account for prod GAR | Yes |
| `DOCKERHUB_USERNAME` | Docker Hub username (avoids rate limits) | Optional |
| `DOCKERHUB_TOKEN` | Docker Hub access token | Optional |

## Configuration

These values can be adjusted in the workflow file:

| Variable | Default | Description |
|----------|---------|-------------|
| `SOURCE_IMAGES` | `taskcluster/taskcluster taskcluster/websocktunnel` | Images to sync |
| `TAG_PATTERN` | `^v?[0-9]+\.[0-9]+\.[0-9]+(-devel)?$` | Regex for tags to sync |
| `MAX_RECENT_TAGS` | `10` | Max recent tags to check per scheduled run |
| Schedule | Every 30 minutes | Cron schedule for polling |
