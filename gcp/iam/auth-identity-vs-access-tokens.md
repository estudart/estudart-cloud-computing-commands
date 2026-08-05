# Identity tokens vs access tokens (the mix-up that wastes the most time)
<!-- verified: 2026-08 -->

| Token | Command | Used for |
|---|---|---|
| Identity token | `gcloud auth print-identity-token` | Calling Cloud Run / IAP-protected services |
| Access token | `gcloud auth print-access-token` | Calling Google APIs (Vertex AI, Cloud Storage, etc.) |

```
gcloud auth print-identity-token
gcloud auth print-access-token
```

**Why it bites:** using the wrong one doesn't fail loudly and specifically — it gives a plausible-looking auth error either way, so it's easy to go chasing IAM permissions when the real problem is sending an access token to something that wanted an identity token (or vice versa).

## Reading the response code correctly
- **401** → missing token, or a token with the wrong audience. Fix: confirm you're using the right *kind* of token and that its audience matches the callee's URL.
- **403** → the token is valid and Google-recognized, but the identity behind it lacks `roles/run.invoker` (or the equivalent role) on the resource. Fix: an IAM binding, not the token.

These need different fixes — don't start editing IAM bindings on a 401, and don't start regenerating tokens on a 403.
