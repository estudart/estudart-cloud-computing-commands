# Artifact Registry: create a repo, authenticate, push
<!-- verified: 2026-08 -->

## Create a repository (one-time per project/region)
```
gcloud artifacts repositories create <REPO_NAME> \
  --project=<PROJECT> \
  --repository-format=docker \
  --location=<REGION>
```
Example:
```
gcloud artifacts repositories create nike-retail-images \
  --project=nike-retail-prod \
  --repository-format=docker \
  --location=us-central1
```

## Authenticate Docker before pushing
```
gcloud auth configure-docker <REGION>-docker.pkg.dev --quiet
```
Example:
```
gcloud auth configure-docker us-central1-docker.pkg.dev --quiet
```
**Why it bites:** skip this and Docker sends no credentials at all with the push. The error is `Unauthenticated request` — reads exactly like an IAM/permissions problem, but it's purely a local Docker config step, nothing to fix on the GCP side.

## Push
```
docker push <REGION>-docker.pkg.dev/<PROJECT>/<REPO_NAME>/<IMAGE>:<TAG>
```
Example:
```
docker push us-central1-docker.pkg.dev/nike-retail-prod/nike-retail-images/api-backend:a1b2c3d
```
