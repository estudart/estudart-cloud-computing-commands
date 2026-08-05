# Org policies that force a different architecture (not just a permission tweak)
<!-- verified: 2026-08 -->

## `iam.disableServiceAccountKeyCreation`
Blocks creation of SA JSON key files org-wide.
```
gcloud resource-manager org-policies describe iam.disableServiceAccountKeyCreation --project=<PROJECT>
```
**Why it bites:** any CI pipeline designed around mounting a downloaded SA key file simply can't deploy — `gcloud iam service-accounts keys create` fails outright. This isn't a one-line fix; it forces a keyless auth path (Workload Identity Federation, or authenticating CI as a human via OIDC) instead of patching the existing pipeline.

## Domain Restricted Sharing
Refuses any IAM binding with `allUsers` or `allAuthenticatedUsers` as the member.
```
gcloud resource-manager org-policies describe iam.allowedPolicyMemberDomains --project=<PROJECT>
```
**Why it bites:** this rules out the classic load-balancer + IAP pattern outright, since that setup typically needs an `allUsers` binding on the backend before IAP narrows access back down. Direct IAP-on-Cloud-Run (no load balancer — see `gcp/deploy/setup-iap-cloud-run.md`) sidesteps this because it never needs `allUsers` at any point, which is why it's the right call under this policy, not just a simpler alternative.
