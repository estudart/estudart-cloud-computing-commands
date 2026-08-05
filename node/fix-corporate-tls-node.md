# Corporate TLS interception breaks Node but not curl
<!-- verified: 2026-08 -->

On a managed laptop behind corporate TLS inspection, `curl` works (it reads the macOS System keychain) while Node fails with:
```
unable to get local issuer certificate
```
Node ships its own bundled CA list and never looks at the OS keychain.

## Local fix (never bake this into an image)
```
security find-certificate -a -p /Library/Keychains/System.keychain > /tmp/corp_roots.pem
export NODE_EXTRA_CA_CERTS=/tmp/corp_roots.pem
```
**Why it bites:** this is a workstation-only fix for a workstation-only problem — your laptop's network is intercepted, a Cloud Run container's isn't. Putting `NODE_EXTRA_CA_CERTS` or a corporate root cert into a Dockerfile "to be safe" bakes a local certificate into a shipped image for no reason. Leave it out of anything that gets deployed.
