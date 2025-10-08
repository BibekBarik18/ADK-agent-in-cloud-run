# ADK-agent-in-cloud-run

This repository contains a minimal example for packaging an ADK (Agent Development Kit) agent and deploying it to Google Cloud Run using the gcloud CLI and a container image.

The instructions below show how to prepare your environment, build an image from the included `Dockerfile`, and deploy the service to Cloud Run (managed). Examples include PowerShell (recommended for Windows users) and Bash examples.

Reference: ADK Cloud Run deployment docs — https://google.github.io/adk-docs/deploy/cloud-run/#payload

## Quick checklist

- Google Cloud SDK (`gcloud`) installed and authenticated
- Docker installed (or use Cloud Build)
- Cloud Run API (and optionally Cloud Build and Artifact Registry / Container Registry) enabled for your project
- Your agent code is in a directory that contains at least `__init__.py`, `agent.py` (which exposes `root_agent`), and `requirements.txt` (for Python)

## Environment variables

A sample env file is available at `academic_research/.env.example`. Populate the values for your project. Common variables used by ADK and this repo:

- `GOOGLE_CLOUD_PROJECT` - your GCP project id
- `GOOGLE_CLOUD_LOCATION` - e.g. `us-central1`
- `GOOGLE_GENAI_USE_VERTEXAI` - `1` / `0` or `True` / `False`
- `GOOGLE_CLOUD_STORAGE_BUCKET` - optional GCS bucket name
- `USER_ID` - optional user id used by your agent

You can either set these in Cloud Run when deploying or pass them via `--set-env-vars` during `gcloud run deploy`.

## Build and push container image

Two common approaches are shown: use Cloud Build to build & push, or build locally and push to a registry. Below examples assume `SERVICE_NAME` and project variables are set.

PowerShell (Windows) example using Cloud Build:

```powershell
# Set variables (PowerShell)
$env:GOOGLE_CLOUD_PROJECT = "your-gcp-project-id"
$env:GOOGLE_CLOUD_LOCATION = "us-central1"
$env:SERVICE_NAME = "service-name"

# Build and push with Cloud Build to Container Registry (gcr.io)
gcloud builds submit --project $env:GOOGLE_CLOUD_PROJECT --tag "gcr.io/$($env:GOOGLE_CLOUD_PROJECT)/$($env:SERVICE_NAME)"
```

Bash example:

```bash
# Set variables (bash)
export GOOGLE_CLOUD_PROJECT="your-gcp-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export SERVICE_NAME="service-name"

# Build and push
gcloud builds submit --project $GOOGLE_CLOUD_PROJECT --tag gcr.io/$GOOGLE_CLOUD_PROJECT/$SERVICE_NAME
```

If you prefer to build locally and push, you can use `docker build` + `docker push` (make sure to run `gcloud auth configure-docker` first to push to gcr.io).

## Deploy to Cloud Run (gcloud CLI)

The command below deploys the built image to Cloud Run (managed). You can include environment variables using `--set-env-vars` (comma separated), or update them later in the Cloud Console.

PowerShell example (deploy, allow unauthenticated access):

```powershell
# Deploy service (PowerShell)
gcloud run deploy $env:SERVICE_NAME \
  --image "gcr.io/$($env:GOOGLE_CLOUD_PROJECT)/$($env:SERVICE_NAME)" \
  --region $env:GOOGLE_CLOUD_LOCATION \
  --platform managed \
  --allow-unauthenticated \
  --set-env-vars "GOOGLE_GENAI_USE_VERTEXAI=1,GOOGLE_CLOUD_PROJECT=$($env:GOOGLE_CLOUD_PROJECT),GOOGLE_CLOUD_LOCATION=$($env:GOOGLE_CLOUD_LOCATION),USER_ID=u_567,GOOGLE_CLOUD_STORAGE_BUCKET=my-bucket"
```

Bash example:

```bash
gcloud run deploy $SERVICE_NAME \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/$SERVICE_NAME \
  --region $GOOGLE_CLOUD_LOCATION \
  --platform managed \
  --allow-unauthenticated \
  --set-env-vars GOOGLE_GENAI_USE_VERTEXAI=1,GOOGLE_CLOUD_PROJECT=$GOOGLE_CLOUD_PROJECT,GOOGLE_CLOUD_LOCATION=$GOOGLE_CLOUD_LOCATION,USER_ID=u_567,GOOGLE_CLOUD_STORAGE_BUCKET=my-bucket
```

Notes:
- Remove `--allow-unauthenticated` if you want authenticated access only. See the ADK docs section on authenticated access for examples of using identity tokens.
- If you want the ADK dev UI to be served from your container, set the `SERVE_WEB_INTERFACE=True` environment variable (the ADK docs show enabling UI by flag or env var depending on the method used).

## Get the service URL

After deployment you can get the service URL with:

PowerShell:

```powershell
$url = gcloud run services describe $env:SERVICE_NAME --region $env:GOOGLE_CLOUD_LOCATION --platform managed --format "value(status.url)"
Write-Output "Service URL: $url"
```

Bash:

```bash
URL=$(gcloud run services describe $SERVICE_NAME --region $GOOGLE_CLOUD_LOCATION --platform managed --format "value(status.url)")
echo "Service URL: $URL"
```

## Testing your deployed agent

- If you deployed the UI (see `SERVE_WEB_INTERFACE`), open the service URL in a browser. The ADK dev UI will let you select and run your agent.
- For API testing with authentication required, fetch an identity token and use curl.

PowerShell (authenticated request):

```powershell
$token = gcloud auth print-identity-token
Invoke-RestMethod -Method Post -Uri "$url/" -Headers @{ Authorization = "Bearer $token" } -Body (@{ input = "hello" } | ConvertTo-Json) -ContentType 'application/json'
```

Bash (authenticated request):

```bash
TOKEN=$(gcloud auth print-identity-token)
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"input":"hello"}' "$URL/"
```

The exact API path depends on how your agent server is implemented. If you deployed with the ADK default server, open the root URL or the UI to confirm behavior. Consult the ADK docs for API endpoints exposed by the ADK server.

## Helpful tips & troubleshooting

- Ensure you set `GOOGLE_CLOUD_PROJECT` via `gcloud config set project <PROJECT_ID>` or environment variable before builds/deploys.
- If you hit permission errors, confirm the account used by `gcloud` has roles/run.admin and roles/iam.serviceAccountUser (or use a service account with the correct roles).
- Check logs in Cloud Run console or with `gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=$SERVICE_NAME" --limit 50 --project $GOOGLE_CLOUD_PROJECT`.
- If using large dependencies, prefer Cloud Build so the image is built in the cloud.

## Links

- ADK Cloud Run docs (source for payload, env vars, and deploy options): https://google.github.io/adk-docs/deploy/cloud-run/#payload
- gcloud run deploy docs: https://cloud.google.com/sdk/gcloud/reference/run/deploy

