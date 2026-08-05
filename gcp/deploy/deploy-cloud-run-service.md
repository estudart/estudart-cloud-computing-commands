# Deploy a Cloud Run service — full flow: build → tag → push → deploy
<!-- verified: 2026-08 -->

End-to-end sequence for shipping a new revision, in the order you actually run it. Placeholders first, then the same sequence filled in with fake data (project `nike-retail-prod`, region `us-central1`, repo `nike-retail-images`, service `api-backend`).

## The full sequence, copy-paste block
```
docker build --platform linux/amd64 -t <IMAGE> .
TAG=$(git rev-parse --short HEAD)$([ -n "$(git status --porcelain)" ] && echo -dirty)
docker tag <IMAGE> <REGION>-docker.pkg.dev/<PROJECT>/<REPO>/<IMAGE>:$TAG
gcloud auth configure-docker <REGION>-docker.pkg.dev --quiet
docker push <REGION>-docker.pkg.dev/<PROJECT>/<REPO>/<IMAGE>:$TAG
gcloud run deploy <SERVICE> \
  --image=<REGION>-docker.pkg.dev/<PROJECT>/<REPO>/<IMAGE>:$TAG \
  --project=<PROJECT> --region=<REGION>
```
Filled example:
```
docker build --platform linux/amd64 -t api-backend .
TAG=$(git rev-parse --short HEAD)$([ -n "$(git status --porcelain)" ] && echo -dirty)
docker tag api-backend us-central1-docker.pkg.dev/nike-retail-prod/nike-retail-images/api-backend:$TAG
gcloud auth configure-docker us-central1-docker.pkg.dev --quiet
docker push us-central1-docker.pkg.dev/nike-retail-prod/nike-retail-images/api-backend:$TAG
gcloud run deploy api-backend \
  --image=us-central1-docker.pkg.dev/nike-retail-prod/nike-retail-images/api-backend:$TAG \
  --project=nike-retail-prod --region=us-central1
```

The rest of this file is the same sequence broken into steps, with the failure mode for each one.

## 1. Build for the right architecture
```
docker build --platform linux/amd64 -t <IMAGE> .
docker image inspect <IMAGE> --format '{{.Architecture}}'
```
Example:
```
docker build --platform linux/amd64 -t api-backend .
docker image inspect api-backend --format '{{.Architecture}}'
```
Expect `amd64` in the output before moving on.

**Why it bites:** Cloud Run is amd64-only. An arm64 image — the default when building on Apple Silicon without `--platform` — deploys "successfully" and only fails at runtime, with the generic `container failed to start and listen on the port defined by the PORT environment variable`. That message has two other unrelated causes below, so check architecture first since it's the cheapest to rule out.

## 2. Tag with the git SHA, never `latest`
```
TAG=$(git rev-parse --short HEAD)$([ -n "$(git status --porcelain)" ] && echo -dirty)
docker tag <IMAGE> <REGION>-docker.pkg.dev/<PROJECT>/<REPO>/<IMAGE>:$TAG
```
Example:
```
TAG=$(git rev-parse --short HEAD)$([ -n "$(git status --porcelain)" ] && echo -dirty)
docker tag api-backend us-central1-docker.pkg.dev/nike-retail-prod/nike-retail-images/api-backend:$TAG
```
The `-dirty` suffix marks a build made from an uncommitted tree, so it never silently claims to match a commit it doesn't.

**Why it bites:** a Cloud Run revision pins the image *digest* it resolved at deploy time — it does not keep tracking the `latest` tag afterward. If every push is tagged `latest`, a list of revisions tells you nothing about what code is actually running in each one.

## 3. Authenticate Docker to Artifact Registry (one-time per machine, per region)
```
gcloud auth configure-docker <REGION>-docker.pkg.dev --quiet
```
Example:
```
gcloud auth configure-docker us-central1-docker.pkg.dev --quiet
```
**Why it bites:** skip this and Docker pushes with no credentials at all. The error is `Unauthenticated request` — reads exactly like an IAM/permissions problem, but it's purely a local Docker config step, nothing to fix on the GCP side. (Repo creation itself lives in `docker/push-artifact-registry.md`.)

## 4. Push
```
docker push <REGION>-docker.pkg.dev/<PROJECT>/<REPO>/<IMAGE>:<TAG>
```
Example:
```
docker push us-central1-docker.pkg.dev/nike-retail-prod/nike-retail-images/api-backend:$TAG
```

## 5. Deploy — and know this is create-or-update
```
gcloud run deploy <SERVICE> \
  --image=<REGION>-docker.pkg.dev/<PROJECT>/<REPO>/<IMAGE>:<TAG> \
  --project=<PROJECT> --region=<REGION>
```
Example:
```
gcloud run deploy api-backend \
  --image=us-central1-docker.pkg.dev/nike-retail-prod/nike-retail-images/api-backend:$TAG \
  --project=nike-retail-prod --region=us-central1
```
**Why it bites:** this command preserves every setting you don't explicitly pass — env vars, scaling, service account all stay whatever they were before. A "just ship the image" deploy script quietly leaves env vars/scaling/SA under whatever was last set through the console. Fine, until someone assumes the deploy script is the full source of truth for the service's config and it isn't.

## 6. Updating env vars afterward: replace vs merge
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
**Why it bites:** using `--set-env-vars` to add just one new var wipes everything else, including things like `DATABASE_URL`. The resulting failure is the same generic PORT-listen error from step 1 — it says nothing about a missing env var. Default to `--update-env-vars` unless you deliberately want to replace the whole set.

## 7. Never set PORT yourself
Cloud Run injects `PORT` into the container and expects the app to listen on it. Setting it manually — as an env var or baked into the image — causes a collision and the container is rejected, producing that same generic PORT-listen message a third time.

**Why it bites:** that message now has three unrelated causes (wrong architecture, wiped env vars, manual PORT). When it shows up, rule out all three in order rather than committing to one theory.

## 8. Background work after the response is sent
Cloud Run throttles CPU to near-zero once the HTTP response is sent, unless the service has always-allocated CPU or `--min-instances >= 1`. Anything kicked off in the background — fire-and-forget logging, cleanup, webhooks — can silently die mid-execution.

**Why it bites:** these failures never reach the client, since the response already returned 200. The only trace is in the logs, and only if the code got far enough to log something before being throttled.

## 9. Where to actually see what happened
```
gcloud run services logs read <SERVICE> --region=<REGION> --limit=100 --project=<PROJECT>
```
Example:
```
gcloud run services logs read api-backend --region=us-central1 --limit=100 --project=nike-retail-prod
```
