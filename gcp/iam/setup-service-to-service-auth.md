# Service-to-service auth: ADC + the run.invoker binding direction
<!-- verified: 2026-08 -->

## Getting an identity token in code (identical locally and in Cloud Run)
```js
const auth = new GoogleAuth();
const client = await auth.getIdTokenClient(audience);
```
`audience` = the callee's base URL, e.g. `https://<CALLEE_SERVICE_URL>`.

This resolves Application Default Credentials automatically — the metadata server when running inside Cloud Run, your `gcloud` session when running on a laptop. Same code path either way, only the credential source differs, which is why testing locally against a deployed backend is genuinely representative of prod behavior.

**Why it bites:** `~/.config/gcloud/application_default_credentials.json` with `"type": "authorized_user"` is a cached refresh token, not a short-lived session. That's why calls from a laptop keep working for months after the last `gcloud auth login` — it doesn't mean auth is misconfigured, it means the cache hasn't expired.

## Granting one service permission to call another
```
gcloud run services add-iam-policy-binding <CALLEE_SERVICE> \
  --project=<PROJECT> --region=<REGION> \
  --member="serviceAccount:<CALLER_SERVICE_ACCOUNT>" \
  --role="roles/run.invoker"
```
Example:
```
gcloud run services add-iam-policy-binding api-backend \
  --project=nike-retail-prod --region=us-central1 \
  --member="serviceAccount:web-frontend-sa@nike-retail-prod.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```

**Why it bites (this is the whole point of this file):**
- **Resource = the service being called. Member = the caller.** It reads backwards from how you'd say it out loud ("frontend calls backend" → the binding goes on the *backend*, naming the *frontend's* SA as the member). Swap them and the command still succeeds — it's a silent no-op on the wrong resource, not an error. You get 403s forever and a clean `gcloud` output telling you nothing's wrong.
- **IAM propagation takes minutes.** Retrying immediately after applying the binding looks exactly like a real permission failure. Wait a few minutes before concluding the binding didn't work or chasing an unrelated hypothesis (wrong region, wrong SA, etc.) — this alone can cost an afternoon.

## Default compute service accounts may have almost nothing
If an org policy disables automatic Editor grants, the default compute SA (`<PROJECT_NUMBER>-compute@developer.gserviceaccount.com`) may start with only `roles/logging.logWriter`. Calling Vertex AI then fails specifically on `aiplatform.endpoints.predict`.

Fix: grant the specific role needed, and prefer a dedicated per-service SA over the shared default going forward:
```
gcloud projects add-iam-policy-binding <PROJECT> \
  --member="serviceAccount:<SERVICE_ACCOUNT>" \
  --role="roles/aiplatform.user"
```
Example:
```
gcloud projects add-iam-policy-binding nike-retail-prod \
  --member="serviceAccount:api-backend-sa@nike-retail-prod.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"
```
