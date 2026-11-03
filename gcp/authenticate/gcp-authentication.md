# Authenticate with Google Cloud

This guide configures both authentication layers commonly needed during local development:

1. **Google Cloud CLI credentials** — used by `gcloud`, `gcloud storage`, and related commands.
2. **Application Default Credentials (ADC)** — used by Python Google Cloud client libraries.

These credential stores are separate. A successful `gcloud auth login` does not necessarily authenticate Python code.

## Standard local authentication

Set the project once at the beginning. Replace the placeholder before running the commands.

```bash
GCP_PROJECT_ID="<YOUR_GCP_PROJECT_ID>"
```

Remove any service-account override from the current terminal session:

```bash
unset GOOGLE_APPLICATION_CREDENTIALS
```

Authenticate the Google Cloud CLI:

```bash
gcloud auth login --force
```

Set the default project used by `gcloud`:

```bash
gcloud config set project "${GCP_PROJECT_ID}"
```

Authenticate Python and other Google client libraries through ADC:

```bash
gcloud auth application-default login \
  --project="${GCP_PROJECT_ID}"
```

Explicitly set the project used for API quota and billing attribution:

```bash
gcloud auth application-default set-quota-project \
  "${GCP_PROJECT_ID}"
```

## Verify the configuration

Show the authenticated CLI accounts and active project:

```bash
gcloud auth list
gcloud config get-value account
gcloud config get-value project
```

Verify that the CLI can access Cloud Storage:

```bash
gcloud storage buckets list --project="${GCP_PROJECT_ID}"
```

Verify ADC without printing the access token:

```bash
gcloud auth application-default print-access-token >/dev/null \
  && echo "Application Default Credentials: OK"
```

Verify ADC from Python:

```bash
python -c 'import google.auth; from google.auth.transport.requests import Request; credentials, project = google.auth.default(); credentials.refresh(Request()); print({"project": project, "authenticated": credentials.valid})'
```

When the project uses `uv`, run the same check inside its environment:

```bash
uv run python -c 'import google.auth; from google.auth.transport.requests import Request; credentials, project = google.auth.default(); credentials.refresh(Request()); print({"project": project, "authenticated": credentials.valid})'
```

Expected result:

```text
{'project': '<YOUR_GCP_PROJECT_ID>', 'authenticated': True}
```

## Short combined login

Recent versions of the Google Cloud CLI can update CLI credentials and ADC in one authentication flow:

```bash
gcloud auth login --force --update-adc
gcloud config set project "${GCP_PROJECT_ID}"
gcloud auth application-default set-quota-project "${GCP_PROJECT_ID}"
```

The explicit two-login sequence is easier to diagnose because it makes the two credential stores visible.

## Authenticate without automatically opening a browser

Use this when working in a remote terminal or when the automatic browser flow does not work:

```bash
gcloud auth login --no-launch-browser
gcloud auth application-default login \
  --no-launch-browser \
  --project="${GCP_PROJECT_ID}"
```

Open the displayed URL in a trusted browser and paste the resulting authorization code into the terminal.

## Switch accounts

List credentialed accounts:

```bash
gcloud auth list
```

Select a different account for CLI commands:

```bash
gcloud config set account "<USER_EMAIL>"
```

ADC must be authenticated separately if Python should also use the new identity:

```bash
gcloud auth application-default login "<USER_EMAIL>" \
  --project="${GCP_PROJECT_ID}"
```

## Troubleshooting

### Python uses an unexpected service account

Check whether a credential file is overriding ADC:

```bash
printenv GOOGLE_APPLICATION_CREDENTIALS
```

Remove the override from the current shell and authenticate ADC again:

```bash
unset GOOGLE_APPLICATION_CREDENTIALS
gcloud auth application-default login \
  --project="${GCP_PROJECT_ID}"
```

### The quota-project command returns a permission error

The authenticated identity needs `serviceusage.services.use` on the selected project. A commonly used predefined role containing that permission is:

```text
roles/serviceusage.serviceUsageConsumer
```

Authentication proves identity; it does not grant IAM permissions. Ask a project administrator for the narrow role required by the operation.

### `gcloud` works but Python receives `401` or cannot find credentials

Authenticate ADC explicitly:

```bash
gcloud auth application-default login \
  --project="${GCP_PROJECT_ID}"
```

Then rerun the ADC verification command.

### Python works but `gcloud` uses the wrong account

Select the intended CLI account:

```bash
gcloud auth list
gcloud config set account "<USER_EMAIL>"
```

## Revoke local credentials

Revoke CLI credentials for one account:

```bash
gcloud auth revoke "<USER_EMAIL>"
```

Revoke Application Default Credentials:

```bash
gcloud auth application-default revoke
```

## Security notes

- Never commit access tokens, service-account keys, or ADC files.
- Do not print access tokens into shared terminals, screenshots, tickets, or logs.
- Prefer user ADC for local development and Workload Identity Federation for CI/CD.
- Avoid long-lived service-account JSON keys whenever possible.
- Use the narrowest IAM permissions needed by the application or operator.

## Example project

For NeoAgentic ConfHub development:

```bash
GCP_PROJECT_ID="oa-neo-agentic-ai-dv"
```
