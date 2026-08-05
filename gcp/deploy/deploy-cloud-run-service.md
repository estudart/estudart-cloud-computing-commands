# Deploy a Cloud Run service without wiping its config
<!-- verified: 2026-08 -->

## The core gotcha: deploy is create-or-update, and only touches what you pass
```
gcloud run deploy <SERVICE> \
  --image=<REGION>-docker.pkg.dev/<PROJECT>/<REPO>/<IMAGE>:<TAG> \
  --project=<PROJECT> --region=<REGION>
```
Example:
```
gcloud run deploy api-backend \
  --image=us-central1-docker.pkg.dev/nike-retail-prod/nike-retail-images/api-backend:a1b2c3d \
  --project=nike-retail-prod --region=us-central1
```
**Why it bites:** this command preserves every setting you don't explicitly pass — env vars, scaling, service account, all stay whatever they were before. A "just ship the image" deploy script quietly leaves env vars/scaling/SA under whatever was last set through the console. Fine, until someone assumes the deploy script is the full source of truth for the service's config and it isn't.

## Updating env vars: replace vs merge
```
# Wipes every existing env var and sets only these:
gcloud run services update <SERVICE> --region=<REGION> --set-env-vars=<KEY>=<VALUE>

# Merges — keeps existing vars, adds/overwrites only these:
gcloud run services update <SERVICE> --region=<REGION> --update-env-vars=<KEY>=<VALUE>
```
Example:
```
gcloud run services update api-backend --region=us-central1 \
  --update-env-vars=FEATURE_FLAG_NEW_CHECKOUT=true
```
**Why it bites:** using `--set-env-vars` to add just one new var wipes everything else, including things like `DATABASE_URL`. The resulting failure is `container failed to start and listen on the port defined by the PORT environment variable` — which says nothing about a missing env var. Default to `--update-env-vars` unless you deliberately want to replace the whole set.

## Never set PORT yourself
Cloud Run injects `PORT` into the container and expects your app to listen on it. Setting it manually — as an env var or baked into the image — causes a collision and the container is rejected, producing the exact same generic "failed to start and listen on PORT" message as above.

**Why it bites:** this is the *third* unrelated cause of that identical error message (the others being the wiped env var above, and the architecture mismatch below). When you see it, rule out all three before committing to one theory.

## Build for the right architecture (bites constantly on Apple Silicon)
Cloud Run is amd64-only. An arm64 image built on an Apple Silicon Mac without the platform flag deploys "successfully" and then fails at runtime with the same PORT error.
```
docker build --platform linux/amd64 -t <IMAGE_TAG> .
docker image inspect <IMAGE_TAG> --format '{{.Architecture}}'
```
Example:
```
docker build --platform linux/amd64 -t api-backend:a1b2c3d .
docker image inspect api-backend:a1b2c3d --format '{{.Architecture}}'
```
Expect `amd64` in the output before you push — `arm64` means the flag was forgotten.

## Tag with the git SHA, never `latest`
```
docker build --platform linux/amd64 -t <REGION>-docker.pkg.dev/<PROJECT>/<REPO>/<IMAGE>:$(git rev-parse --short HEAD) .
```
Append `-dirty` to the tag if `git status --porcelain` isn't empty, so a build never claims to match a commit it doesn't.

**Why it bites:** a Cloud Run revision pins the image *digest* it resolved at deploy time — it does not keep tracking the `latest` tag afterward. If everything is tagged `latest`, a list of revisions tells you nothing about what code is actually running in each one.

## Background work after the response is sent
Cloud Run throttles CPU to near-zero once the HTTP response is sent, unless the service has always-allocated CPU or `--min-instances >= 1`. Anything kicked off in the background — fire-and-forget logging, cleanup, webhooks — can silently die mid-execution.

**Why it bites:** these failures never reach the client, since the response already returned 200. The only trace is in the logs, and only if the code got far enough to log something before being throttled.

## Where to actually see what happened
```
gcloud run services logs read <SERVICE> --region=<REGION> --limit=100 --project=<PROJECT>
```
