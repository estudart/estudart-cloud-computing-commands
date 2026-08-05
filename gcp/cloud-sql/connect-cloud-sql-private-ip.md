# Cloud SQL: private IP vs the /cloudsql socket vs Auth Proxy
<!-- verified: 2026-08 -->

Three different connection paths that look interchangeable and aren't.

## Private IP over direct VPC egress — raw TCP, no IAM involved
```
psql "host=<PRIVATE_IP> dbname=<DB_NAME> user=<DB_USER>"
```
**Why it bites:** this is a raw TCP connection over the VPC — IAM is never consulted, so `roles/cloudsql.client` is *not* required, and granting it does nothing for this path. If a connection fails here, the fix is networking (VPC peering, firewall, direct egress config), not IAM.

## The `/cloudsql/<connection-name>` unix socket — goes through the IAM-gated connector
```
psql "host=/cloudsql/<CONNECTION_NAME> dbname=<DB_NAME> user=<DB_USER>"
```
Connection name format: `<PROJECT>:<REGION>:<INSTANCE>`. Example: `nike-retail-prod:us-central1:nike-retail-db`

**Why it bites:** this form *does* require `roles/cloudsql.client` on the calling identity. Moving a service from the private-IP path to this socket form (e.g. between environments) silently breaks if that role was never granted — it was never needed on the other path.

## Cloud SQL Auth Proxy from a laptop
```
cloud-sql-proxy <CONNECTION_NAME>
```
Example:
```
cloud-sql-proxy nike-retail-prod:us-central1:nike-retail-db
```
**Why it bites:** the Auth Proxy is not a VPC tunnel. If the instance only has a private IP and the laptop isn't on the VPC, the proxy can't reach it — a bastion host or VPN is a prerequisite, not a proxy configuration option.
