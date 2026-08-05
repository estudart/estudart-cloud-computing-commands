# Put a Cloud Run service behind IAP (no load balancer)
<!-- verified: 2026-08 -->

Direct IAP-on-Cloud-Run. No load balancer, no OAuth client to create by hand — both steps below are GA, no `beta` needed.

## 1. Enable the API
```
gcloud services enable iap.googleapis.com --project=<PROJECT>
```
Example:
```
gcloud services enable iap.googleapis.com --project=nike-retail-prod
```

## 2. OAuth consent screen (once per project)
IAP won't turn on without one. Check first — a project-wide brand may already exist:
```
gcloud iap oauth-brands list --project=<PROJECT>
```
If empty, create it in the console (UI-only, no gcloud command for this part):
1. Console → **APIs & Services** → **OAuth consent screen**
2. **User type**: Internal — restricts sign-in to your Cloud Identity org automatically, no scopes or test users needed
3. **App name**: anything internal-facing
4. **User support email** and **Developer contact email**: yours
5. Save

## 3. Turn IAP on for the service
```
gcloud run services update <SERVICE> --project=<PROJECT> --region=<REGION> --iap
```
Example:
```
gcloud run services update web-frontend --project=nike-retail-prod --region=us-central1 --iap
```
This sets the annotation `run.googleapis.com/iap-enabled: 'true'` and should auto-grant `roles/run.invoker` to IAP's own service agent. If it doesn't prompt/grant automatically, do it explicitly:
```
gcloud run services add-iam-policy-binding <SERVICE> \
  --project=<PROJECT> --region=<REGION> \
  --member="serviceAccount:service-<PROJECT_NUMBER>@gcp-sa-iap.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```
Example:
```
gcloud run services add-iam-policy-binding web-frontend \
  --project=nike-retail-prod --region=us-central1 \
  --member="serviceAccount:service-748213590264@gcp-sa-iap.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```
Note the resource here is the service itself — IAP is the caller now, so the service's own door is what's being unlocked.

## 4. Let a domain through
```
gcloud iap web add-iam-policy-binding \
  --resource-type=cloud-run --service=<SERVICE> \
  --project=<PROJECT> --region=<REGION> \
  --member="domain:<DOMAIN>" --role="roles/iap.httpsResourceAccessor"
```
Example:
```
gcloud iap web add-iam-policy-binding \
  --resource-type=cloud-run --service=web-frontend \
  --project=nike-retail-prod --region=us-central1 \
  --member="domain:nike.com" --role="roles/iap.httpsResourceAccessor"
```

**Why it bites (this whole setup):**
- Steps 3 and 4 are separate on purpose. Step 3 turns the lock on; step 4 hands out keys. Do only step 3 and *nobody* gets in — including you.
- `domain:<DOMAIN>` only works if the project sits under that domain's Cloud Identity org. Verify before troubleshooting anything else:
  ```
  gcloud projects describe <PROJECT> --format='value(parent.type,parent.id)'
  gcloud organizations list
  ```
- Once IAP is on, a valid Cloud Run invoker *identity token* stops working for browser-style access — IAP wants its own session cookie instead. `gcloud run services proxy` and a plain `curl` with a bearer token no longer get you in the way they used to before IAP.
- IAP injects `X-Goog-Authenticated-User-Email` and `X-Goog-IAP-JWT-Assertion` on every request. For anything security-sensitive, verify the JWT against Google's public keys — only trust the plain email header because the service is unreachable except through IAP.
- `--no-iap` turns enforcement back off, but leaves the service still requiring *some* auth — expect a bare 403 in a browser, not open access, until the invoker binding is also loosened.
- No OAuth client to create manually. IAP provisions its own — the old "create an OAuth client" flow belongs to the load-balancer + IAP path, whose admin APIs are being phased out.

## Verify enforcement
```
curl -D - <SERVICE_URL>
```
Anonymous request should return `302`, redirecting to `accounts.google.com` with `scope=openid+email`.
